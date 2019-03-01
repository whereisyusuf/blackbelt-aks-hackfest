# Securing Your App

While a public app is great - however we will often come across scenarios where we want a private access to my application. 

## Making the API private

We will update the API service to have an internal load balancer. 

In the yaml file - we have made the following changes 

- Added an annotation for using Azure Internal Load Balancer. 
- Added a specific IP for the load balancer - (Note: Do ensure that this IP does not conflict with any already given IP addresses)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  labels:
    name: api
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  loadBalancerIP: 10.1.0.250
  ports:
  - name: http
    port: 3000
    targetPort: 3000
  selector:
    name: heroes-api
```
We also need to change the web pod to update the service name instead of the public IP address
```yaml
env:
        - name:  API
          value:  http://api:3000/
```
Finally update the app

```azurecli-interactive
kubectl apply -f blackbelt-aks-hackfest/labs/helper-files/heroes-web-ilb-api.yaml
```


Output:

```json
{
  "id": "/subscriptions/4f48eeae-9347-40c5-897b-46af1b8811ec/resourcegroups/myResourceGroup/providers/Microsoft.ContainerService/managedClusters/myK8sCluster",
  "location": "eastus",
  "name": "myK8sCluster",
  "properties": {
    "accessProfiles": {
      "clusterAdmin": {
        "kubeConfig": "..."
      },
      "clusterUser": {
        "kubeConfig": "..."
      }
    },
    "agentPoolProfiles": [
      {
        "count": 1,
        "dnsPrefix": null,
        "fqdn": null,
        "name": "myK8sCluster",
        "osDiskSizeGb": null,
        "osType": "Linux",
        "ports": null,
        "storageProfile": "ManagedDisks",
        "vmSize": "Standard_D2_v2",
        "vnetSubnetId": null
      }
    ],
    "dnsPrefix": "myK8sClust-myResourceGroup-4f48ee",
    "fqdn": "myk8sclust-myresourcegroup-4f48ee-406cc140.hcp.eastus.azmk8s.io",
    "kubernetesVersion": "1.11.3",
    "linuxProfile": {
      "adminUsername": "azureuser",
      "ssh": {
        "publicKeys": [
          {
            "keyData": "..."
          }
        ]
      }
    },
    "provisioningState": "Succeeded",
    "servicePrincipalProfile": {
      "clientId": "e70c1c1c-0ca4-4e0a-be5e-aea5225af017",
      "keyVaultSecretRef": null,
      "secret": null
    }
  },
  "resourceGroup": "myResourceGroup",
  "tags": null,
  "type": "Microsoft.ContainerService/ManagedClusters"
}
```

You can now confirm the upgrade was successful with the `az aks show` command.

```azurecli-interactive
az aks show --name $CLUSTER_NAME --resource-group $NAME --output table
```

Output:

```json
Name          Location    ResourceGroup    KubernetesVersion    ProvisioningState    Fqdn
------------  ----------  ---------------  -------------------  -------------------  ----------------------------------------------------------------
myK8sCluster  eastus     myResourceGroup  1.11.3                Succeeded            myk8sclust-myresourcegroup-3762d8-2f6ca801.hcp.eastus.azmk8s.io
```

## Attribution:
Content originally created by @gabrtv et al. from [this](https://docs.microsoft.com/en-us/azure/aks/upgrade-cluster) Azure Doc
