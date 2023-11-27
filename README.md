# Introduction



## Getting Started

This repo contains the code and instructions to deploy Azure Container Storage using CLI and deploy workloads.

### Installation

```bash
# Upgrade to the latest version of the aks-preview cli extension by running the following command.
az extension add --upgrade --name aks-preview

# Add or upgrade to the latest version of k8s-extension by running the following command.
az extension add --upgrade --name k8s-extension

# set subscription context
az account set --subscription <subscription-id>

# register resoure providers
az provider register --namespace Microsoft.ContainerService --wait 
az provider register --namespace Microsoft.KubernetesConfiguration --wait

# create a resource group
az group create --name <resource-group-name> --location <location>
az aks create -n <cluster-name> -g <resource-group-name> --node-vm-size Standard_D4s_v3 --node-count 3 --enable-azure-container-storage <storage-pool-type>

#display available storage pools
kubectl get sp –n acstor

#display storage classes
kubectl get sc
```



## Demo Jupytehub
create config.yaml with content

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
    Authenticator:
      admin_users:        
        - adminuser
```
install jupyterhub

```bash
choco install kubernetes-helm
```

```bash
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update
```
```bash
helm upgrade --cleanup-on-fail --install jhub1 jupyterhub/jupyterhub --namespace jhub1 --create-namespace --values config.yaml
```
```bash
  kubectl get service --namespace jhub1
```

Port forward to connect to the service
```bash
kubectl --namespace=jhub1 port-forward service/proxy-public 8080:http
```
Note: If the port-forward doesnt work from the portal CLI, you can try using local terminal

From the browser:
Log on to:

http://localhost:8080/

Generate the token: Tokens are sent to the Hub for verification. The Hub replies with a JSON model describing the authenticated user.

http://localhost:8080/hub/token

Update the value of the token in the 'api_token' parameter in the python script (user_creation.py)

Run python script
py user_creation.py


Run these commands in your cluster to get the pods and pvcs'
```bash
kubectl get pvc -n jhub1
kubectl get pods -n jhub1
```
## Resources
Provide additional resource like installing choco
