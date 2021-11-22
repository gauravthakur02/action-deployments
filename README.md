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
---
2. **Create Kubernetes manifest files in `/kubernetes`:**
* Create `blue-deploy.yaml` file:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
      matchLabels:
        app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers: 
        - name: nginx
          image: demogaurav.azurecr.io/blue-nginx:1
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 200m
              memory: 200Mi
            limits:
              cpu: 500m
              memory: 500Mi
```
* Create `green-deploy.yaml` file:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-2
spec:
  replicas: 3
  selector:
      matchLabels:
        app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers: 
        - name: nginx
          image: demogaurav.azurecr.io/green-nginx:1
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 200m
              memory: 200Mi
            limits:
              cpu: 500m
              memory: 500Mi
```
* Create `service.yaml` file:
```yml
apiVersion: v1
kind: Service
metadata: 
  name: nginx
  labels: 
    app: nginx
spec:
  ports:
    - name: http
      port: 80
      targetPort: 80
  selector: 
    app: nginx
  type: LoadBalancer
```
---
3. **Create GitHub Actions workflow in `/.github/workflows`:**
```yml
# This is a basic workflow to help you get started with Actions
name: Blue-Green-strategy

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the main branch
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

env:
  ACR_NAME: demogaurav.azurecr.io
  ACR_REPO_NAME: demogaurav.azurecr.io/docker-java-app
  ARTIFACT_NAME: docker-java-app
  RESOURCE_GROUP: Gaurav-RG
  AKS_CLUSTER_NAME: Gaurav-AKS-cluster
  
# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  deployapp:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2

      # This action can be used to set cluster context before other actions like azure/k8s-deploy, azure/k8s-create-secret or any kubectl commands (in script) can be run subsequently in the workflow.
      - name: Set cluster context
        uses: azure/aks-set-context@v1
        with:
          creds: '${{ secrets.AZURE_CREDENTIALS }}'
          cluster-name: ${{ env.AKS_CLUSTER_NAME }}
          resource-group: ${{ env.RESOURCE_GROUP }}
      
      # Runs a set of commands using the runners shell
      - name: Deploy app
        uses: azure/k8s-deploy@v1.3
        with:
          namespace: default
          manifests: |
            ./kubernetes/service.yaml
            ./kubernetes/blue-deploy.yaml
          images: |
            demogaurav.azurecr.io/green-nginx:1
          strategy: blue-green
          traffic-split-method: pod
          action: deploy  #deploy is the default; we will later use this to promote/reject

  approveapp:
    runs-on: ubuntu-latest
    needs: deployapp
    environment: akspromotion
    steps:
      - run: echo asked for approval

  promotereject:
    runs-on: ubuntu-latest
    needs: approveapp
    steps:
      - uses: actions/checkout@v2

      - name: Set cluster context
        uses: azure/aks-set-context@v1
        with:
          creds: '${{ secrets.AZURE_CREDENTIALS }}'
          cluster-name: ${{ env.AKS_CLUSTER_NAME }}
          resource-group: ${{ env.RESOURCE_GROUP }}

      - name: Promote App
        uses: azure/k8s-deploy@v1.3
        if: ${{ success() }}
        with:
          namespace: default
          manifests: |
            ./kubernetes/service.yaml
            ./kubernetes/green-deploy.yaml
          images: |
            demogaurav.azurecr.io/green-nginx:1
          strategy: blue-green
          traffic-split-method: pod
          action: promote  #deploy is the default; we will later use this to promote/reject

      - name: Reject App
        uses: azure/k8s-deploy@v1.3
        if: ${{ failure() }}
        with:
          namespace: default
          manifests: |
            ./kubernetes/service.yaml
            ./kubernetes/blue-deploy.yaml
          images: |
            demogaurav.azurecr.io/green-nginx:1
          strategy: blue-green
          traffic-split-method: pod
          action: reject  #deploy is the default; we will later use this to promote/reject
```
![Blue-Green Deployment Workflow]()
---
4. **Run the `blue-green-strategy` workflow:**
* ![deployapp]()
    * ![pod1]()
    * ![service1]()
    * ![app1]()
* ![approveapp]()
* ![promotereject]()
    * ![pod2]()
    * ![service2]()
    * ![app2]()
---
