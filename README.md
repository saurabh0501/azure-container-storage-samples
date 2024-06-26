# Introduction



## Getting Started with Azure Container Storage

This repo contains the code and instructions to deploy Azure Container Storage using CLI and deploy Jupyter & Kafka workloads.

### Pre-requisites (if not running on Cloud Shell)
* Install [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli#install-or-update)
* Install [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-kubectl-binary-with-curl-on-windows)

### Installation

```bash
# Upgrade to the latest version of the aks-preview cli extension by running the following command.
az extension add --upgrade --name aks-preview

# Add or upgrade to the latest version of k8s-extension by running the following command.
az extension add --upgrade --name k8s-extension

# Set subscription context
az account set --subscription <subscription-id>

# Register resoure providers
az provider register --namespace Microsoft.ContainerService --wait 
az provider register --namespace Microsoft.KubernetesConfiguration --wait

# Create a resource group
az group create --name <resource-group-name> --location <location>

# Create an AKS cluster with Azure Container Storage extension enabled
az aks create -n <cluster-name> -g <resource-group-name> --node-vm-size Standard_D4s_v3 --node-count 3 --enable-azure-container-storage azureDisk

# Display available storage pools, one was created when Azure Container Storage was enabled
kubectl get sp –n acstor

# Display storage classes - there should be a storage class created that corresponds to the storage pool
kubectl get sc
```

## Demo Jupytehub

### Pre-requisites
* Install [Python](https://www.python.org/downloads/windows/) (if not running on Cloud Shell)
* Install [requests](https://pypi.org/project/requests/) Python library (if not running on Cloud Shell)
* Install [Chocolatey](https://chocolatey.org/install) to install helm (if not running on Cloud Shell)

### Deployment

1. Create config.yaml file to specify that each single user that will be created will get provisioned 1 Gi of storage, from a StorageClass created via Azure Container Storage.

```bash
code config.yaml
```
```bash
singleuser:
  storage:
    capacity: 1Gi
    dynamic:
      storageClass: acstor-azuredisk
hub:
  config:
    JupyterHub:
      admin_access: false
    Authenticator:
      admin_users:
        - admin
```
2. Install jupyterhub


```bash
(if not running on Cloud Shell) choco install kubernetes-helm
```

```bash
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update
```
This is installing JupyterHub using the information provided on config.yaml, and creating a namespace (jhub1) where the demo resources will sit.
```bash
helm upgrade --cleanup-on-fail --install jhub1 jupyterhub/jupyterhub --namespace jhub1 --create-namespace --values config.yaml
```
```bash
kubectl get service --namespace jhub1
```

3. Port forward to connect to the service
```bash
kubectl --namespace=jhub1 port-forward service/proxy-public 8080:http
```
Note: If the port-forward doesn't work from CloudShell, you can try using your computer's local terminal

#### Create additional user sessions
1. From the browser - log on to: http://localhost:8080/ using the credentials from the config.yaml file (username: admin).

2. Generate the token from http://localhost:8080/hub/token. Tokens are sent to the Hub for verification. The Hub replies with a JSON model describing the authenticated user.

3. Update the value of the token in the 'api_token' parameter in the Python script (user_creation.py)

4. Run python script
```bash
py user_creation.py
```

5. Run these commands in your cluster to get the pods and PVCs
```bash
kubectl get pvc -n jhub1
kubectl get pods -n jhub1
```

## Demo Kafka
Kafka Message producer is a small Go lang application to generate X number of messages and ingest to Kafka. It's using confluent-kafka-go library. The kafka-x-messages-producer.yaml file requires some environment variables and the kafka user1 password as a secret. Environment variables are: NUM_MESSAGES, KAFKA_TOPIC, KAFKA_ADDR, KAFKA_USER and KAFKA_PASSWORD(as kubernetes secret).

```bash
export AZURE_SUBSCRIPTION_ID=<your_subscriptionID>
export AZURE_RESOURCE_GROUP="aks-kafka-san"
export AZURE_CLUSTER_NAME="aks-kafka-san"

# existing grafana and azure monitoring
grafanaId=$(az grafana show --name grafana-azure-ase --resource-group aks-azure --query id --output tsv)
azuremonitorId=$(az resource show --resource-group aks-azure --name grafana-azure-ase --resource-type "Microsoft.Monitor/accounts" --query id --output tsv)

az group create --name $AZURE_RESOURCE_GROUP --location australiaeast
az aks create -g $AZURE_RESOURCE_GROUP -n $AZURE_CLUSTER_NAME --generate-ssh-keys \
       --node-count 3 \
       --os-sku AzureLinux \
       --enable-azure-container-storage elasticSan \
       --node-vm-size standard_d8s_v5 \
       --max-pods=250 \
       --enable-azure-monitor-metrics \
       --azure-monitor-workspace-resource-id $azuremonitorId \
       --grafana-resource-id $grafanaId 
```

## Deploy Kafka from Bitnami using Helm

```bash
# Install Kafka using HELM
helm upgrade kafka --namespace kafka --create-namespace \
  --set controller.replicaCount=3 \
  --set global.storageClass=acstor-elasticSan \
  --set controller.heapOpts="-Xmx2048m -Xms2048m" \
  --set controller.persistence.enabled=True \
  --set controller.persistence.size=1Ti \
  --set controller.logPersistence.enabled=True \
  --set controller.logPersistence.size=100Gi \
  --set controller.resources.limits.cpu=2 \
  --set controller.resources.limits.memory=4Gi \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=2Gi \
  --set metrics.jmx.enabled=True \
  oci://registry-1.docker.io/bitnamicharts/kafka
```

## Create secret for default kafka user1 generated on the installation

```bash
kubectl create secret generic kafka-user-password \
--namespace=kafka \
--from-literal=password="$(kubectl get secret kafka-user-passwords --namespace kafka -o jsonpath='{.data.client-passwords}' | base64 -d | cut -d , -f 1)"
```

## Create Deployment to Produce X messages

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-x-messages-producer
  namespace: kafka
spec:
  replicas: 100
  selector:
    matchLabels:
      app: kafka-x-messages-producer
  template:
    metadata:
      labels:
        app: kafka-x-messages-producer
    spec:
      containers:
      - name: kafka-x-messages-producer
        image: docker.io/jorgearteiro/kafka-x-messages-producer:0.9.0
        command: ["./main"]
        env:
        - name: NUM_MESSAGES
          value: "10000000"
        - name: KAFKA_TOPIC
          value: "orders"
        - name: KAFKA_ADDR
          value: "kafka.kafka.svc.cluster.local:9092"
        - name: KAFKA_USER
          value: "user1"
        - name: KAFKA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: kafka-user-password
              key: password
        resources:
          limits:
            cpu: "0.06"
            memory: "64Mi"
          requests:
            cpu: "0.01"
            memory: "32Mi"
```

## Creating Elastic San Pools using CRDs

```bash
kubectl apply -f - <<EOF
apiVersion: containerstorage.azure.com/v1alpha1
kind: StoragePool
metadata:
  name: san1
  namespace: acstor
spec:
  poolType:
    elasticSan: {}
  resources:
    requests: {"storage": 1Ti}
EOF
```


## Elasticsearch demo

This repo contains the code and instructions to deploy Azure Container Storage using CLI and deploy a ElasticSearch workload.

You can read more about Azure Container Storage [here](https://learn.microsoft.com/en-us/azure/storage/container-storage/container-storage-introduction) - the industry’s first platform-managed container native storage service in the public cloud, providing highly scalable, cost-effective persistent volumes, built natively for containers.

## Getting Started with Azure Container Storage

### Pre-requisites
If you are running in CloudShell, you do not need to install Azure CLI or Kubectl, but we recommend running on your local terminal as there is a JupyterHub specific step that doesn't work on CloudShell.
* Install [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli#install-or-update)
* Install [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-kubectl-binary-with-curl-on-windows)

### Installation

```bash
# Upgrade to the latest version of the aks-preview cli extension by running the following command.
az extension add --upgrade --name aks-preview

# Add or upgrade to the latest version of k8s-extension by running the following command.
az extension add --upgrade --name k8s-extension

# Set subscription context
az account set --subscription <subscription-id>

# Register resoure providers
az provider register --namespace Microsoft.ContainerService --wait 
az provider register --namespace Microsoft.KubernetesConfiguration --wait

# Create a resource group
az group create --name <resource-group-name> --location <location>

# Create an AKS cluster with Azure Container Storage extension enabled. This will create a StoragePool of type Azure Disk by default. If you want to update the defaults (pool name, pool size or SKU), you can do so by using the parameters here: https://learn.microsoft.com/en-us/azure/storage/container-storage/container-storage-aks-quickstart#create-a-new-aks-cluster-and-install-azure-container-storage
az aks create -n <cluster-name> -g <resource-group-name> --node-vm-size Standard_D8ds_v4 --node-count 3 --enable-azure-container-storage azureDisk --node-count 5 --nodepool-name systempool

# Connect to the AKS cluster
az aks get-credentials --resource-group <resource-group-name> --name <cluster-name>


 # Add a user nodepool
```bash
  az aks nodepool add --cluster-name <cluster-name> --mode User --name espoolz1 --node-vm-size Standard_D8ds_v4 --resource-group <resource-group-name> --zones 1 --enable-cluster-autoscaler --max-count 12 --min-count 5 --node-count 5--labels app=es
```

 # Label the user node pool
```bash
 az aks nodepool update --resource-group <resource-group-name> --cluster-name <cluster-name> --name espoolz1 --labels acstor.azure.com/io-engine=acstor
```

## Elastic Search Cluster Installation 

### Prepare The Cluster 
We will use the "acstor-azuredisk" storage class 


## We will use helm to install the ElasticSearch cluster, we will rely on the ElasticSearch chart provided by bitnami as its the easiest one to navigate. 

## add the bitnami repository
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```
## Get the values file we'll need to update 
We will create our own values file (there is a sample (values_acs.yaml) in this repo you can use) where we can 

1. Adjust the affinity and taints to match our node pools 
2. adjust the number of replicas and the scaling parameters for master, data, and coordinating, and ingestion nodes
3. configure the storage class 
4. optionally make the elastic search service accessible using a load balancer 
5. enable HPA for all the nodes 
   

### ElasticSearch Cluster Deployment

Now that we have configured the charts, we are ready to deploy the ES cluster 

```bash
# Create the namespace
kubectl create namespace elasticsearch

# Install elastic search using the values file 
helm install elasticsearch-v1 bitnami/elasticsearch -n elasticsearch --values values_acs.yaml

# Validate the installation, it will take around 5 minutes for all the pods to move to a 'READY' state 
watch kubectl get pods -o wide -n elasticsearch


# Check the service so we can access elastic search, note the "External-IP" 
kubectl get svc -n elasticsearch elasticsearch-v1
```

#
```bash
kubectl -n elasticsearch port-forward svc/elasticsearch-v1 9200:9200 &
curl http://$SERVICE_IP:9200/
```

## Create an index 
```bash
##create an index called "acstor" with 3 replicas 
curl -X PUT "http://$esip:9200/acstor" -H "Content-Type: application/json" -d '{
  "settings": {
    "number_of_replicas": 3
  }
}'

##test the index 
curl -X GET "http://$esip:9200/acstor"
```


## Ingest some data in elasticsearch using python 

# Install docker

## download the Dockerfile and ingest_logs.py and build the docker image

# create an azure container registry (acr) from the portal

# Change the registry name to match yours
```bash
#Point the folder to the one having docker file
az acr login --name IgniteCR
az acr update --name IgniteCR --anonymous-pull-enabled
docker build -t ignitecr.azurecr.io/my-ingest-image:1.0 .
docker push ignitecr.azurecr.io/my-ingest-image:1.0 
```

#run the job (remember to change the image name to yours in ingest-job.yaml) also change the parallelism and completions to match your needs
```bash
cd ..
kubectl apply -f ingest-job.yaml
```
# to verify the job is running
```bash
kubectl get pods -l app=log-ingestion 
kubectl logs -l app=log-ingestion -f 
```

# watch the elastic search pods being scaled out 
```bash
watch kubectl get pods -n elasticsearch
kubectl get hpa -n elasticsearch 
```


## Cassandra demo

## Getting Started with Azure Container Storage

This repo contains the code and instructions to deploy Azure Container Storage using CLI and deploy Jupyter & Kafka workloads.

### Pre-requisites (if not running on Cloud Shell)
* Install [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli#install-or-update)
* Install [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-kubectl-binary-with-curl-on-windows)

### Installation

```bash
# Upgrade to the latest version of the aks-preview cli extension by running the following command.
az extension add --upgrade --name aks-preview

# Add or upgrade to the latest version of k8s-extension by running the following command.
az extension add --upgrade --name k8s-extension

# Set subscription context
az account set --subscription <subscription-id>

# Register resoure providers
az provider register --namespace Microsoft.ContainerService --wait 
az provider register --namespace Microsoft.KubernetesConfiguration --wait

# Create a resource group
az group create --name <resource-group-name> --location <location>

# Create an AKS cluster with Azure Container Storage extension enabled
az aks create -n <cluster-name> -g <resource-group-name> --node-vm-size Standard_D4s_v3 --node-count 3 --enable-azure-container-storage ephemeralDisk

kubectl apply -f acstor-storagepool.yaml

kubectl describe sp ephemeraldisk -n acstor

kubectl get sc

kubectl apply -f acstor-pvc.yaml

kubectl describe pvc ephemeralpvc

helm install cassandra bitnami/cassandra -n cassandra --create-namespace --set global.storageClass=acstor-ephemeraldisk --set nodeSelector."acstor\.azure\.com/io-engine"=acstor --set dbUser.user=admin,dbUser.password=password --set replicaCount=12 --set persistence.size=32Gi --set metrics.enabled=true --wait --debug  --timeout 15m

kubectl describe pod cassandra-0 -n cassandra

kubectl describe pvc data-cassandra-0 -n cassandra

kubectl describe sp ephemeraldisk -n acstor
