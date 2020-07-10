# Create and deploy an ASP.NET core web app that reads secret settings from Azure Key Vault using AppAuthentication libary

## 1. Create an ASP.NET core web app
* If you use Visual Studio, follow instructions at [Quickstart: Use Visual Studio to create your first ASP.NET Core web app](https://docs.microsoft.com/en-us/visualstudio/ide/quickstart-aspnet-core?view=vs-2019)
* If you are using Command Line tools, follow instructions at [Tutorial: Get started with ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/getting-started/?view=aspnetcore-3.1&tabs=linux)

## 2. Add Azure Key Vault and AppAuthentication libraries to the web app
Add the following NuGet packages to your web app:
```dotnetcli
dotnet add package Microsoft.Azure.KeyVault
dotnet add package Microsoft.Azure.Services.AppAuthentication
dotnet add package Microsoft.Extensions.Configuration.AzureKeyVault
```

## 3. Add the following code to Program.cs file to initalize Key Vault as a configuration provider

You can refer to the Program.cs file in this repository

```C#
 public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                //code for Azure Key Vault config provider
                .ConfigureAppConfiguration((ctx, builder) =>
                {
                    var keyVaultEndpoint = GetKeyVaultEndpoint();
                    if (!string.IsNullOrEmpty(keyVaultEndpoint))
                    {
                        var azureServiceTokenProvider = new AzureServiceTokenProvider();
                        var keyVaultClient = new KeyVaultClient(
                            new KeyVaultClient.AuthenticationCallback(
                                azureServiceTokenProvider.KeyVaultTokenCallback));
                        builder.AddAzureKeyVault(
                        keyVaultEndpoint, keyVaultClient, new DefaultKeyVaultSecretManager());
                    }
                })
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                });

        //private method for getting Key Vault end point
        private static string GetKeyVaultEndpoint() => Environment.GetEnvironmentVariable("KEYVAULT_ENDPOINT");
```

## 4. Add code in view to demonstrate the web app can read a secret from Key Vault. Add the setting to appsettings.json file
Add the following code in Index.cshtml. you can refer to /aspnet-k8s/views/Index.cshtml as reference

```C#
<h1>@config["kvsecret"];</h1> 
```

Add the kvsecret as a setting in appsettings.json. Initialize with empty value.
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*",
  "kvsecret": ""
}
```
Run the web app to make sure there is no build errors. You should see the homepage displaying empty kvsecret value.

## 5. Add docker support to the web app
* If you use Visual Studio, follow instructions at [Container Tools in Visual Studio](https://docs.microsoft.com/en-us/visualstudio/containers/overview?view=vs-2019)
* Otherwise, follow instructions at [Dockerize an ASP.NET Core application](https://docs.docker.com/engine/examples/dotnetcore/)

If you build docker image and run locally, you should see the homepage displaying empty kvsecret value.

## 6. Push docker image to an Azure Container Registry
* Create an Azure Container Register if you don't have one already: [Quickstart: Create a private container registry using the Azure CLI](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli)

* List the docker image for your web app
```dotnetcli
docker images
```

* Tag the image so you can push it to your ACR later
```dotnetcli
docker tag <your_image_name> <your_ACR_name>.azurecr.io/<your_image_name>:v1
```

* Login to your ACR if you haven't done so in the create step
```azurecli
az acr login --name <registry-name>
```

* Push your docker image
```dotnetcli
docker push <your_ACR_name>.azurecr.io/<your_image_name>:v1
```

* you can go to Azure portal to check if your image is present in the Azure Container Registry

## 6. Deploy AKS cluster. Grant access to container registry
* Follow instructions at [Quickstart: Deploy an Azure Kubernetes Service cluster using the Azure CLI
](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough#create-aks-cluster) to create an AKS cluster
* The following command in the instruction above creates the rbac-enabled AKS with ACR attached to it
```dotnetcli
	az aks create --resource-group <YOUR_RESROUCEGROUP_NAME> --name <YOUR_CLUSTER_NAME> --node-count 1 --enable-addons monitoring --generate-ssh-keys --enable-managed-identity --enable-rbac --attach-acr <YOUR_AZURECONTAINERRESGISTRY_NAME>
```

## 7. Create Azure Key Vault and add a secret with name 'kvsecret' to it
* Follow instructions at [Quickstart: Set and retrieve a secret from Azure Key Vault using the Azure portal](https://docs.microsoft.com/en-us/azure/key-vault/secrets/quick-create-portal) to create a Key Vault
* Follow [add-a-secret-to-key-vault](https://docs.microsoft.com/en-us/azure/key-vault/secrets/quick-create-portal#add-a-secret-to-key-vault) to add a secret named 'kvsecret', which is what's used in the web app

## 8. Create AKS pod identity
```dotnetcli
	az identity create -g <YOUR_AKSSERVICE_RESOURCEGROUP> -n <YOUR_POD_IDENTITY_NAME> -o json
```

## 9. Assign pod identity reader access to Key Vault resource group and secret management access to Key Vault
```dotnetcli
az role assignment create --role Reader --assignee <YOUR_POD_IDENTITY_CLIENTID> -scope /subscriptions/<YOUR_SUBSCRIPTION_ID>/resourcegroups/<YOUR_KEYVAULT_RESOURCEGROUP>

az keyvault set-policy --name "<YOUR_KEYVAULT_NAME>" --object-id "<YOUR_POD_IDENTITY_PRINCIPALID>" --secret-permissions get list
```

## 10. Assign cluster Managed Identity permissions to AKS cluster resource group for MIC to work

If you followed previous instructions to create the AKS cluster, there should be a Managed Identity resource named <YOUR_CLUSTERNAME-agentpool> in the resource group <MC_yourAKSclustername_region>. Get the Object ID for this Managed Identity - you can copy from the portal essentials bar.

Run the following commands for assigning permissions for the identity to access this cluster's VMSS and other identities

```dotnetcli
az role assignment create --role "Managed Identity Contributor" --assignee <CLUSTERIDENTITY_OBJECT_ID> --scope /subscriptions/<YOUR_SUBSCRIPTION_id>/resourcegroups/<YOUR_AKS_CLUSTER_RESOURCEGROUP_STARTINGWITH_MC>

az role assignment create --role "Virtual Machine Contributor" --assignee <CLUSTERIDENTITY_OBJECT_ID> --scope /subscriptions/<YOUR_SUBSCRIPTION_id>/resourcegroups/<YOUR_AKS_CLUSTER_RESOURCEGROUP_STARTINGWITH_MC>

```

## 11. Assign cluster Managed Identity permissions to pod identity
Use the same Object ID obtained from previous step to grand cluster Managed Identity permissions to access application Pod's identity. If you followed the instructions above to create the Pod identity, it should be in the same resource group as your AKS service (i.e. NOT the MC_ resource group).

```AzureCLI
az role assignment create --role "Managed Identity Operator" --assignee <CLUSTERIDENTITY_CLIENT_ID> --scope /subscriptions/<YOUR_SUBSCRIPTION_ID>/resourcegroups/<YOUR_PODIDENTITY_RESOURCEGROUP>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<YOUR_POD_IDENTITY_NAME>

```

## 12. Add aad-pod-identity controller pods to cluster

Connect to your AKS cluster if you have not done so already:

```AzureCLI
az aks get-credentials --resource-group <YOUR_AKSSERVICE_RESOURCEGROUP> --name <YOUR_AKSCLUSTER_NAME>
```

Add [Managed Identity Controller](https://github.com/Azure/aad-pod-identity#managed-identity-controller) and [Node Managed Identity](https://github.com/Azure/aad-pod-identity#node-managed-identity):

```AzureCLI
kubectl create -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml 
```

## 13. Create Kubernetes Deployment file
Download the /aspnet-k8s/deploy_withPodIdentity.yml file in this repository. Edit the parameters using your environment information.

## 14. Deploy the app
```dotnetcli
kubectl apply -f ./deploy_withPodIdentity.yml
```

## 15. Access the service
Get the ClusterIP by running:
```dotnetcli
kubectl get services
```

Open the IP in a web browser, you should see the secret from Key vault showing up in the Homepage.

## 16. Debug
If the Pod identity is not working correctly, get the logs from NMI pods
1. get the names of the NMI pods
```dotnetcli
kubectl get pods
```
2. Choose one MNI pod and get log files
```dotnetcli
kubectl logs <NAME_OF_NMIPODS>
```