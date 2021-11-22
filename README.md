# Guide to "blue-green" and "canary" deployments using GitHub Actions

## What is Blue/Green deployment strategy in Kubernetes?
>Blue/Green deployments are a form of progressive delivery where a new version of the application is deployed while the old version still exists. The two versions coexist for a brief period of time while user traffic is routed to the new version, before the old version is discarded (if all goes well).

---
## What is Canary deployment strategy in Kubernetes?
>Canary deployment strategy involves deploying new versions of an application next to stable production versions to see how the canary version compares against the baseline before promoting or rejecting the deployment. 

---
### **This is a simple tutorial on how to do [Blue/Green Deployment] on Kubernetes.**

### Prerequisites
* Any Kubernetes cluster 1.3+ should work. Create an [AKS Cluster](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough) using [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/):
```
az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 1 --enable-addons monitoring --generate-ssh-keys
```
* Application source code
* Docker image of the application
* Kubernetes manifest files _(blue-deploy.yaml, green-deploy.yaml, service.yaml)_

---
1. **Create Docker image from Dockerfile in `/nginx-html`:**
>Edit the `index.html` file to get different webpage message on each new docker image created from the below `Dockerfile`. (For this demo, I have created two docker images "_demo.azurecr.io/blue-nginx:1_" and "_demo.azurecr.io/green-nginx:1_")
```Dockerfile
FROM ubuntu

RUN apt-get update

RUN apt-get install nginx -y

COPY index.html /var/www/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Build docker image using below command, e.g.:
```
docker build -t demo.azurecr.io/blue-nginx:1 .
```
```
docker build -t demo.azurecr.io/green-nginx:1 .
```
Upload the docker image to ACR/DockerHub (we are using ACR):
```
docker push demo.azurecr.io/blue-nginx:1
```
```
docker push demo.azurecr.io/green-nginx:1
```
