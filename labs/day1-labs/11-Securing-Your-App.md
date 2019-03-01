# Securing Your App

While a public app is great - however we will often come across scenarios where we want to secure the app behind a WAF.  

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

If we run the following command

```cli 
yusuf@Azure:~$ kubectl get pods,svc
```

We will see the following output 
```cli
NAME                                     READY     STATUS    RESTARTS   AGE
pod/heroes-api-deploy-6bfd578b87-t9k8v   1/1       Running   0          19h
pod/heroes-db-deploy-6bffff4ff5-l9nhh    1/1       Running   0          20h
pod/heroes-web-deploy-6f5866cbdd-c4hrw   1/1       Running   0          19h

NAME                 TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
service/api          LoadBalancer   10.0.1.175   10.1.0.250    3000:30508/TCP   19h
service/kubernetes   ClusterIP      10.0.1.1     <none>        443/TCP          21h
service/mongodb      ClusterIP      10.0.1.73    <none>        27017/TCP        20h
service/web          LoadBalancer   10.0.1.52    10.1.0.251    8080:31142/TCP   19h
```

## Adding Application Gateway with WAF

Create an Application Gateway

```cli
az network application-gateway create --name yr-aks-appgateway --resource-group yr-aks-rg --capacity 1 --frontend-port 80 --http-settings-port 8080 --http-settings-protocol Http --public-ip-address yr-aks-appgateway-ip --servers 10.1.0.251 --sku Standard_Small --subnet appg-subnet --vnet-name yr-aks-rg-vnet
```

Output:

```json
{
  "applicationGateway": {
    "authenticationCertificates": [],
    "backendAddressPools": [
      {
        "etag": "W/\"4fbdb53c-4d10-4e20-8add-4b65641c4185\"",
        "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/backendAddressPools/appGatewayBackendPool",
        "name": "appGatewayBackendPool",
        "properties": {
          "backendAddresses": [
            {
              "ipAddress": "10.1.0.251"
            }
          ],
          "provisioningState": "Succeeded",
          "requestRoutingRules": [
            {
              "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/requestRoutingRules/rule1",
              "resourceGroup": "yr-aks-rg"
            }
          ]
        },
        "resourceGroup": "yr-aks-rg",
        "type": "Microsoft.Network/applicationGateways/backendAddressPools"
      }
    ],
    "backendHttpSettingsCollection": [
      {
        "etag": "W/\"4fbdb53c-4d10-4e20-8add-4b65641c4185\"",
        "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/backendHttpSettingsCollection/appGatewayBackendHttpSettings",
        "name": "appGatewayBackendHttpSettings",
        "properties": {
          "connectionDraining": {
            "drainTimeoutInSec": 1,
            "enabled": false
          },
          "cookieBasedAffinity": "Disabled",
          "pickHostNameFromBackendAddress": false,
          "port": 8080,
          "protocol": "Http",
          "provisioningState": "Succeeded",
          "requestRoutingRules": [
            {
              "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/requestRoutingRules/rule1",
              "resourceGroup": "yr-aks-rg"
            }
          ],
          "requestTimeout": 30
        },
        "resourceGroup": "yr-aks-rg",
        "type": "Microsoft.Network/applicationGateways/backendHttpSettingsCollection"
      }
    ],
    "frontendIPConfigurations": [
      {
        "etag": "W/\"4fbdb53c-4d10-4e20-8add-4b65641c4185\"",
        "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/frontendIPConfigurations/appGatewayFrontendIP",
        "name": "appGatewayFrontendIP",
        "properties": {
          "httpListeners": [
            {
              "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/httpListeners/appGatewayHttpListener",
              "resourceGroup": "yr-aks-rg"
            }
          ],
          "privateIPAllocationMethod": "Dynamic",
          "provisioningState": "Succeeded",
          "publicIPAddress": {
            "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/publicIPAddresses/yr-aks-appgateway-ip",
            "resourceGroup": "yr-aks-rg"
          }
        },
        "resourceGroup": "yr-aks-rg",
        "type": "Microsoft.Network/applicationGateways/frontendIPConfigurations"
      }
    ],
    "frontendPorts": [
      {
        "etag": "W/\"4fbdb53c-4d10-4e20-8add-4b65641c4185\"",
        "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/frontendPorts/appGatewayFrontendPort",
        "name": "appGatewayFrontendPort",
        "properties": {
          "httpListeners": [
            {
              "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/httpListeners/appGatewayHttpListener",
              "resourceGroup": "yr-aks-rg"
            }
          ],
          "port": 80,
          "provisioningState": "Succeeded"
        },
        "resourceGroup": "yr-aks-rg",
        "type": "Microsoft.Network/applicationGateways/frontendPorts"
      }
    ],
    "gatewayIPConfigurations": [
      {
        "etag": "W/\"4fbdb53c-4d10-4e20-8add-4b65641c4185\"",
        "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/gatewayIPConfigurations/appGatewayFrontendIP",
        "name": "appGatewayFrontendIP",
        "properties": {
          "provisioningState": "Succeeded",
          "subnet": {
            "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/virtualNetworks/yr-aks-rg-vnet/subnets/appg-subnet",
            "resourceGroup": "yr-aks-rg"
          }
        },
        "resourceGroup": "yr-aks-rg",
        "type": "Microsoft.Network/applicationGateways/gatewayIPConfigurations"
      }
    ],
    "httpListeners": [
      {
        "etag": "W/\"4fbdb53c-4d10-4e20-8add-4b65641c4185\"",
        "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/httpListeners/appGatewayHttpListener",
        "name": "appGatewayHttpListener",
        "properties": {
          "frontendIPConfiguration": {
            "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/frontendIPConfigurations/appGatewayFrontendIP",
            "resourceGroup": "yr-aks-rg"
          },
          "frontendPort": {
            "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/frontendPorts/appGatewayFrontendPort",
            "resourceGroup": "yr-aks-rg"
          },
          "protocol": "Http",
          "provisioningState": "Succeeded",
          "requestRoutingRules": [
            {
              "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/requestRoutingRules/rule1",
              "resourceGroup": "yr-aks-rg"
            }
          ],
          "requireServerNameIndication": false
        },
        "resourceGroup": "yr-aks-rg",
        "type": "Microsoft.Network/applicationGateways/httpListeners"
      }
    ],
    "operationalState": "Running",
    "probes": [],
    "provisioningState": "Succeeded",
    "redirectConfigurations": [],
    "requestRoutingRules": [
      {
        "etag": "W/\"4fbdb53c-4d10-4e20-8add-4b65641c4185\"",
        "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/requestRoutingRules/rule1",
        "name": "rule1",
        "properties": {
          "backendAddressPool": {
            "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/backendAddressPools/appGatewayBackendPool",
            "resourceGroup": "yr-aks-rg"
          },
          "backendHttpSettings": {
            "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/backendHttpSettingsCollection/appGatewayBackendHttpSettings",
            "resourceGroup": "yr-aks-rg"
          },
          "httpListener": {
            "id": "/subscriptions/967b4ee2-f688-4848-939e-8f7e81e738fe/resourceGroups/yr-aks-rg/providers/Microsoft.Network/applicationGateways/yr-aks-appgateway/httpListeners/appGatewayHttpListener",
            "resourceGroup": "yr-aks-rg"
          },
          "provisioningState": "Succeeded",
          "ruleType": "Basic"
        },
        "resourceGroup": "yr-aks-rg",
        "type": "Microsoft.Network/applicationGateways/requestRoutingRules"
      }
    ],
    "resourceGuid": "bdcacb78-a088-48e2-838d-22be24f192e3",
    "rewriteRuleSets": [],
    "sku": {
      "capacity": 1,
      "name": "Standard_Small",
      "tier": "Standard"
    },
    "sslCertificates": [],
    "urlPathMaps": []
  }
}
```

You can now confirm that the application is working on the app gateway url by pasting its public IP (without port 8080) to check if the app is working as expected. .


## Attribution:
Content originally created by @whereisyusuf et al. from [this](https://github.com/Azure/blackbelt-aks-hackfest/blob/master/labs/day1-labs/10-cluster-upgrading.md) 
