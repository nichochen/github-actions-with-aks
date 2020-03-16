

# Containerizing and deploying appication with GitHub Actions & Azure Kubernetes Service

This demo is going to show you how to containerize and deploy a simple applitoin with CICD tool GtiHub Acions.

## The App
The app used in the demo is the lovely HTML5 mini game Clumsy Bird. Fork the game to your own GitHub account.

    https://github.com/ellisonleao/clumsy-bird


## Docker file
The Dockerfile to containerize the application.

    FROM nginx
    COPY . /usr/share/nginx/html


## GitHub Actions Workflow
Following is the workflow definition. Please replace the `REPLACE_WITH_YOUR_UNIQUE_ID`.

    on: [push]
    
    jobs:
      build:
        runs-on: ubuntu-latest
        steps:
        - uses: actions/checkout@master
        
        - uses: Azure/docker-login@v1
          with:
            login-server: REPLACE_WITH_YOUR_UNIQUE_ID.azurecr.io
            username: ${{ secrets.REGISTRY_USERNAME }}
            password: ${{ secrets.REGISTRY_PASSWORD }}
          
        - run: |
            docker build . -t REPLACE_WITH_YOUR_UNIQUE_ID.azurecr.io/clumsy-bird:${{ github.sha }}
            docker push REPLACE_WITH_YOUR_UNIQUE_ID.azurecr.io/clumsy-bird:${{ github.sha }} 
            
        - name: Setup kubectl
          uses: azure/setup-kubectl@v1
          
        - name: Kubernetes set context
          uses: Azure/k8s-set-context@v1
          with: 
            kubeconfig: ${{secrets.KUBECONFIG}}
        
        - run: |
            kubectl run clumsy-bird --image REPLACE_WITH_YOUR_UNIQUE_ID.azurecr.io/clumsy-bird:${{ github.sha }} -o yaml --dry-run|kubectl apply -n dev -f - 
            kubectl -n dev expose deploy clumsy-bird --port 80  --type LoadBalancer -o yaml --dry-run|kubectl apply -n dev -f - 
    
## Create container registry and Kubernetes on Azure
I created the container registry and Kubernetes cluster on Azure with following script.

    # generate a unique name, please don't include `-` and `-`.
    name="demo${RANDOM}"

    # create resource group
    az group create -n $name -l eastasia
    
    # create and setup a private container registry on Azure
    az acr create -g $name -n $name --sku basic
    az acr update -n $name --admin-enabled 
    # retre
    az acr credential show -n $name
    
    # create a kubernetes cluster on Azure with Azure Kubernetes Service (AKS)
    az aks create -g $name -n dev -c 1 --attach-acr $name
    
    # install Kubernetes client, if not yet installed
    # sudo az aks install-cli
    
    # get the Kubernetes connectoin configuration from AKS
    az aks get-credentials -g $name -n dev -f -
    
    # create a namespace for dev
    kubectl create ns dev