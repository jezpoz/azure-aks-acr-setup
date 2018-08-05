# Walkthrough of the steps required to set up a Azure Kubernetes Service (AKS)

## Install the Azure CLI
* [Link to Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* After installing run the command `az login`
* To use a specific subscription:
  * List the subscriptions on your account `az account list --output table`
  * Set the subscription by running `az account set --subscription "[subscription name / subscription id]"`

## Create the resource group
* [Docs](https://docs.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest)
* Run the command `az group create --location "[location]" --name "[name]"`
* For instance: `az group create --location "westeurope" --name "my_resource_group"`

## Create the Azure Kubernetes Service (AKS)
* [Docs](https://docs.microsoft.com/en-us/azure/aks/)
* Run command `az aks create --resource-group [resource group name] --name [cluster name] --node-count [node-count] --enable-addons [addons] --generate-ssh-keys`
* For instance `az aks create --resource-group "my_resource_group" --name "my-kubernetes-cluster" --node-count 1 --enable-addons monitoring --generate-ssh-keys`
  * Name can contain up to 63 characters and can only contain letters, numbers and dashes (-).

## Installing kubernetes
* Run command `az aks install-cli`
  * This might need elevated permissions

## Connect to the kluster
* Run command `az aks get-credentials --resource-group [resource group name] --name [name of the cluster]`
* For instance `az aks get-credentials --resource-group "my_resource_group" --name "my-kubernetes-cluster"`
  * This might need elevated permissions
* Test that your have the cli installed correctly and connected to the cluster by running the command `kubectl get pods`
  * This should return `No resources found` or a table with pods (if they are allready running)

## Writing charts
* [Docs](https://kubernetes.io/docs/home)
* Please look at `charts/frontend-deployment.yaml` and `charts/frontend-service.yaml`

## Creating your own container registry
* [Docs](https://docs.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest)
* Run command `az acr create --name [container registry name] --resource-group [resource group name] --sku [Basic|Classic|Premium|Standard]`
* For instance `az acr create --name "myContainerRegistry" --resource-group "my-resource-group" --sku Basic`

## Log in to your container registry and push images
* [Docs](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-acr)
* Run command `az acr login --name [container registry name]`

#### Push images
* First, create a docker images that you would like to have in your registry
* Use this command to get the loginServer name for your myContainerRegistry
  * `az acr list --resource-group [resource group name] --query "[].{acrLoginServer:loginServer}" --output table`
  * For me this will be:
  * `az acr list --resource-group "my-resource-group" --query "[].{acrLoginServer:loginServer}" --output table`
* To tag an image, use the `docker tag` command
  * `docker tag [docker image name] [loginServer name]/[docker image name]:[docker image version]`
  * For me this would be:
  * `docker tag frontend myContainerRegistry.azurecr.io/frontend:v1.0`
* Now, push the image to the repository with the `docker push` command
  * `docker push [loginServer name]/[docker image name]:[docker image tag]`
  * For me, this would be
  * `docker push myContainerRegistry.azurecr.io/frontend:v1.0`
* The image should now be pushed to your registry.


## Using the private container registry to pull images to kubernetes deployment
* To create a kubectl secret we need to have this information about our registry:
  * Azure Container Registry Name
  * Service Principal Name (we're going to create that)
  * Azure Container Registry loginServer Name
  * Azure Container Registry app id (We'll fetch that with a command)
  * Service Principal Password (we're going to create that)
  * Service Principal ID (we're going to create that)

Right, let's start.
1. Fetch info about all your registries:
  * `az acr list --output table`
  * Here you will also see the loginServer Name
2. Fetch the registry id
  * `az acr show --name [registry name] --query id --output tsv`
  * For me this will be `az acr show --name "myContainerRegistry" --query id --output tsv`
3. Create a service principal (this will also return the password)
  * `az ad sp create-for-rbac --name [create a name for the service principal] --role Reader --scopes [registry id] --query password --output tsv`
  * For me this will be `az ad sp create-for-rbac --name "my-service-principal" --role Reader --scopes [id from step 2] --query password --output tsv`
4. Get the client id for the service principal
  * `az ad sp show --id http://$SERVICE_PRINCIPAL_NAME --query appId --output tsv`
  * For me this will be `az ad sp show --id http://my-service-principal --query appId --output tsv`
5. When you have all of that, we're going to run `kubectl create secret` command
  * `kubectl create secret docker-registry [secret name] --docker-server [acr-login-server] --docker-username [service-principal-ID] --docker-password [service-principal-password]`
  * For me this will be `kubectl create secret docker-registry acr-auth --docker-server "myContainerRegistry.azurecr.io" --docker-username [ID from step 4] --docker-password [password from step 3]`
6. Now you have created a secret, you can run `kubectl get secrets` to see details about your secret

## Use secret in deployment to pull from private container registry
Change from this:
```
apiVersion: extensions/v1beta1
kind: Deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - image: [public-image-name:image-tag]
          name: frontend
          ports:
          - containerPort: 8080
            protocol: TCP
```
Into this:
```
apiVersion: extensions/v1beta1
kind: Deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - image: [registry name].azurecr.io/[image name]:[image tag]
          name: frontend
          ports:
          - containerPort: 8080
            protocol: TCP
      imagePullSecrets:
      - name: [kubectl secret name]
```

### Apply your deployment and create a service
```
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: web
```
### Azure will automatically assign a loadblancer an external ip. But it will take a while.
Run `kubectl get services` to find out what the services external ip will be.
You can use `kubectl get services -w` (with watch, so it watches for changes)
