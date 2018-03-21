# Intro to Containers on Azure
This lab aims to show how you can quickly deploy container workloads to Azure.

# What is it?

This intro lab serves to guide you on two ways you can deploy a container on Azure (there are more than this...), namely:
*     Deploy a container on an Azure Container Instances (ACI) [ACI](https://azure.microsoft.com/en-us/services/container-instances/)
*     Deploy a managed Kubernetes cluster on Azure using Azure Container Service (AKS) [AKS](https://docs.microsoft.com/en-us/azure/aks/) and deploy our container onto it
*     Push your containers to a private registry using Azure Container Registry (ACR) [ACR](https://azure.microsoft.com/en-us/services/container-registry/)
*     Write to Azure Cosmos DB. [Cosmos DB](https://azure.microsoft.com/en-us/services/cosmos-db/) is Microsoft's globally distributed, multi-model database 
*     Use [Application Insights](https://azure.microsoft.com/en-us/services/application-insights/) to track custom events in the container

# Technology used

* Our container contains a swagger enabled API developed in Go which writes a simple order via json to your specified Cosmos DB and tracks custom events via Application Insights.  

# Preparing for this lab

For this Lab you will require:

* Install the Azure CLI 2.0, get it here - https://docs.microsoft.com/en-us/cli/azure/install-azure-cli
* Install Docker, get it here - https://docs.docker.com/engine/installation/
* Install Kubectl, get it here - https://kubernetes.io/docs/tasks/tools/install-kubectl/
* Install Postman, get it here - https://www.getpostman.com - this is optional but useful

When using the Azure CLI, you first have to log in. You do this by running the command below, and follow the instructions. This will guide you to open a browser and enter a code, after which you can enter your credentials to start a session.

```
az login
```
After logging in, if you have more than one subscripton you may need to set the default subscription you wish to perform actions against. To see the current default subscription, you can run az account show.

```
az account set --subscription "<your requried subscription guid>"
```


## 1. Provisioning a Cosmos DB instance

Let's start by creating a Cosmos DB instance in the portal, this is a quick process. Navigate to the Azure portal and create a new Azure Cosmos DB instance, enter the following parameters:

* ID: &lt;unique db instance name&gt;
* API: Select MongoDB as the API as our container API will use this driver
* ResourceGroup: &lt;unique resource group name&gt;
* Location: West Europe

See below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/CosmosDB.png)

Once the DB is provisioned, we need to get the Database Username and Password, these may be found in the Settings --> Connection Strings section of your DB. We will need these to run our container, so copy them for convenient access. See below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/DBKeys.png)

## 2. Provisioning an Application Insights instance

In the Azure portal, select create new Application Insights instance, enter the following parameters:

* Name: &lt;unique instance name&gt;
* Application Type: General
* ResourceGroup: &lt;resource group you created in step 1&gt;
* Location: West Europe

See below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/ApplicationInsights.png)

Once Application Insights is provisioned, we need to get the Instrumentation key, this may be found in the Overview section. We will need this to run our container, so copy it for convenient access. See below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/AppKey.png)

## 3. Provisioning an Azure Container Registry instance

If you would like an example of how to setup an [Azure Container Registry](https://azure.microsoft.com/en-us/services/container-registry/) instance via ARM, have a look [here](https://github.com/shanepeckham/CADScenario_Recommendations)

Navigate to the Azure Portal and select create new Azure Container Registry, enter the following parameters:

* Registry Name: &lt;unique instance name&gt;
* ResourceGroup: &lt;resource group you created in step 1&gt;
* Location: West Europe
* Admin User: Enable
* SKU: Basic

See below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/ContainerRegistry.png)

## 4. Pull the container to your environment and set the environment keys

Open up your docker command window (if using Windows open it with elevated privileges) and type the following:

``` 
docker pull shanepeckham/go_order_sb
```

We will now test the image locally to ensure that it is working and connecting to our CosmosDB and Application Insights instances. The keys you copied for the DB and App Insights keys are set as environment variables within the container, so we will need to ensure we populate these.

The environment keys that need to be set are as follows:
* DATABASE: <your cosmodb username from step 1>
* PASSWORD: <your cosmodb password from step 1>
* INSIGHTSKEY: <you app insights key from step 2>
* SOURCE: This is a free text field which we will use specify where we are running the container from. I use the values localhost, AppService, ACI and K8 for my tests

So to run the container on your local machine, enter the following command, substituting your environment variable values (if you are running Docker on Windows, omit the 'sudo'):

```
sudo docker run --name go_order_sb -p 8080:8080 -e DATABASE="<your cosmodb username from step 1>" -e PASSWORD="<your cosmodb password from step 1>" -e INSIGHTSKEY="<you app insights key from step 2>" -e SOURCE="localhost"  --rm -i -t shanepeckham/go_order_sb
```
Note, the application runs on port 8080 which we will bind to the host as well. If you are running on Windows, select 'Allow Access' on Windows Firewall.

If all goes well, you should see the application running on localhost:8080, see below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/localrun.png)

Now you can navigate to localhost:8080/swagger and test the api (use Chrome or Firefox). Select the 'POST' /order/ section, select the button "Try it out" and enter some values in the json provided and select "Execute", see below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/swagger.png)

If the request succeeded, you will get a CosmosDB Id returned for the order you have just placed, see below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/swaggerresponse.png)

We can now go and query CosmosDB to check our entry there, in the Azure portal, navigate back to your Cosmos DB instance and go to the section Data Explorer (note, at the time of writing this is in preview so is subject to change). We can now query for the order we placed. A collection called 'orders' will have been created within your database, you can then apply a filter for the id we created, namely:

```
{"id":"5995b963134e4f007bc45447"}
```
See below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/CosmosQuery.png)

## 5. Retag the image and upload it your private Azure Container Registry

Navigate to the Azure Container Registry instance you provisioned within the Azure portal. Click on the *Quick Start* blade, this will provide you with the relevant commands to upload a container image to your registry, see below:

![alt text](https://github.com/shanepeckham/CADScenario_Recommendations/blob/master/images/quicksstartacs.png)

Now we will push the image up to the Azure Container Registry, enter the following (from the quickstart screen):

``` 
docker login <yourcontainerregistryinstance>.azurecr.io

```

To get the username and password, navigate to the *Access Keys* blade, see below:

![alt text](https://github.com/shanepeckham/CADScenario_Recommendations/blob/master/images/acskeys.png)

You will receive a 'Login Succeeded' message. Now type the following:
```
docker tag shanepeckham/go_order_sb <yourcontainerregistryinstance>.azurecr.io/go_order_sb
docker push <yourcontainerregistryinstance>.azurecr.io/go_order_sb
```
Once this has completed, you will be able to see your container uploaded to the Container Registry within the portal, see below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/registryrepo.png)

## 6. Deploy the container to Azure Container Instance

Now we will deploy our container to [Azure Container Instances](https://azure.microsoft.com/en-us/services/container-instances/). 

We will deploy our container instance via the Azure CLI directly.

```
az container create -n go-order-sb -g <myResourceGroup> -e DATABASE=<your cosmodb username from step 1> PASSWORD=<your cosmodb password from step 1> INSIGHTSKEY=<your app insights key from step 2> SOURCE="ACI"--image <yourcontainerregistryinstance>.azurecr.io/go_order_sb:latest --registry-password <your acr admin password> --memory 1 --cpu 1 --dns-name-label <unique dns prefix> --ports 8080
```

You can check the status of the deployment by issuing the container list command:

```
az container show -n go-order-sb -g <myResourceGroup> -o table
```

Once the container has moved to "Succeeded" state you will see your external IP address under the "IP:ports" column, copy this value and navigate to http://yourACIExternalIP:8080/swagger and test your API like before.

## 7. Deploy the container to an Azure Managed Kubernetes Cluster (AKS)

Here we will deploy a Kubernetes cluster using Azure CLI

## Enabling AKS preview for your Azure subscription
While AKS is in preview, creating new clusters may require a feature flag on your subscription. You may request this feature for any number of subscriptions that you would like to use. To check if the feature is enabled, run the following command.

```
az provider show -n Microsoft.ContainerService -o table
```

Use the `az provider register` command to register the AKS provider, if it is not already registered:

```
az provider register -n Microsoft.ContainerService --debug
```

After registering, you are now ready to create a Kubernetes cluster with AKS.

## Create Kubernetes cluster

Use the [az aks create](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az_aks_create) command to create an AKS cluster. The following example creates a cluster named myAKSCluster with three nodes.
This takes about 10 minutes, so grab a cup of coffee, read the documentation: https://docs.microsoft.com/en-us/azure/aks/intro-kubernetes or watch a video: https://channel9.msdn.com/Shows/Azure-Friday/Container-Orchestration-Simplified-with-Managed-Kubernetes-in-Azure-Container-Service-AKS...

```
az aks create --resource-group <myResourceGroup> --location westeurope --name myAKSCluster --node-count 3 --generate-ssh-keys --debug
```

To manage a Kubernetes cluster, use [kubectl][kubectl], the Kubernetes command-line client.

If you're using Azure Cloud Shell, kubectl is already installed. If you want to install it locally, use the [az aks install-cli][az-aks-install-cli] command.


```
az aks install-cli
```

To configure kubectl to connect to your Kubernetes cluster, use the [az aks get-credentials][az-aks-get-credentials] command. This step downloads credentials and configures the Kubernetes CLI to use them.

```
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

To verify the connection to your cluster, use the [kubectl get][kubectl-get] command to return a list of the cluster nodes. Note that this can take a few minutes to appear.

```
kubectl get nodes
```

### Register our Azure Container Registry within Kubernetes

We now want to register our private Azure Container Registry with our Kubernetes cluster to ensure that we can pull images from it. Enter the following within your command window:

```
kubectl create secret docker-registry <yourcontainerregistryinstance> --docker-server=<yourcontainerregistryinstance>.azurecr.io --docker-username=<your acr admin username> --docker-password=<your acr admin password> --docker-email=<youremailaddress.com>
```

To view the Kubernetes dashboard, start the proxy from command line and access the dashboard from your browser on http://localhost:8001/ui/
```
kubectl proxy
```
In the Kubernetes dashboard you should now see this created within the secrets section:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/K8secrets.png)

### Associate the environment variables with container we want to deploy to Kubernetes

We will now deploy our container via a yaml file, which is [here](https://github.com/mpeder/ContainersOnAzure_IntroLab/blob/master/go_order_sb.yaml) but before we do, we need to edit this file to ensure we set our environment variables and ensure that you have set your private Azure Container Registry correctly:

```

spec:
      containers:
      - name: goordersb
        image: <containerregistry>.azurecr.io/go_order_sb
        env:
        - name: DATABASE
          value: "<your cosmodb username from step 1>""
        - name: PASSWORD
          value: "<your cosmodb password from step 1>""
        - name: INSIGHTSKEY
          value: ""<you app insights key from step 2>""
        - name: SOURCE
          value: "K8"
        ports:
        - containerPort: 8080
      imagePullSecrets:
        - name: <yourcontainerregistry>
```

Once the yaml file has been updated, we can now deploy our container. Within the command line enter the following:

```
kubectl create -f ./<your path>/go_order_sb.yaml
```
You should get a success message that a deployment and service has been created. Navigate back to the Kubernetes dashboard and you should see the following:

#### Your deployments running 

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/k8deployments.png)

#### Your three pods

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/k8pods.png)

#### Your service and external endpoint

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/k8service.png)

You can now navigate to http://k8serviceendpoint:8080/swagger and test your API

### View container telemetry in Application Insights

The container we have deployed writes simple events to Application Insights with a time stamp but we could write much richer metrics. Application Insights provides a number of prebuilt dashboards to view application statistics alongside a query tool for getting deep custom insights. For the purposes of this intro we will simply expose the custom events we have tracked, namely the commit to the Azure CosmosDB.

In portal navigate to the Application Insights instance you provisioned and click 'Metrics Explorer', see below:

![alt text](https://github.com/mpeder/ContainersOnAzure_IntroLab/blob/master/images/AppInsightsMetrics.png)
 
Click edit on one of the charts, select a TimeRange and set the Filters to the event names. This will retrieve all of the writes to CosmosDB, see below:

![alt text](https://github.com/mpeder/ContainersOnAzure_IntroLab/blob/master/images/AppInsightsFilter.png)

Finally, for more powerful queries, select the 'Analytics' button, see below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/Analytics.png)

## 8. Clean up the resources you have created

If you dont want to keep using the services you have created during this lab you can now go to the portal and delete the entire resource group. AS you have seen you very quickly create them again :-)


<!-- LINKS - external -->
[azure-vote-app]: https://github.com/Azure-Samples/azure-voting-app-redis.git
[kubectl]: https://kubernetes.io/docs/user-guide/kubectl/
[kubectl-create]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create
[kubectl-get]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get
[kubernetes-documentation]: https://kubernetes.io/docs/home/
[kubernetes-service]: https://kubernetes.io/docs/concepts/services-networking/service/

<!-- LINKS - internal -->
[az-aks-browse]: /cli/azure/aks?view=azure-cli-latest#az_aks_browse
[az-aks-create]: /cli/azure/aks?view=azure-cli-latest#az_aks_create
[az-aks-get-credentials]: /cli/azure/aks?view=azure-cli-latest#az_aks_get_credentials
[az aks install-cli]: /cli/azure/aks?view=azure-cli-latest#az_aks_install_cli
[az-group-create]: /cli/azure/group#az_group_create
[az-group-delete]: /cli/azure/group#az_group_delete
[azure-cli-install]: /cli/azure/install-azure-cli
[aks-tutorial]: ./tutorial-kubernetes-prepare-app.md
