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
az network application-gateway create --name yr-aks-appgateway --resource-group yr-aks-rg --capacity 1 --frontend-port 80 --http-settings-port 8080 --http-settings-protocol Http --public-ip-address yr-aks-appgateway-ip --servers 10.1.0.251 --sku WAF_Medium --subnet appg-subnet --vnet-name yr-aks-rg-vnet
```
The above creates an app gateway with a backend address pool for the WEb. We have to add another address pool for the API

```cli
az network application-gateway address-pool create --name yr-aks-appgateway-api-pool --gateway-name yr-aks-appgateway --resource-group yr-aks-rg --servers 10.1.0.250
```

Create listener 

```cli
az network application-gateway http-listener create --name apibackendListener --frontend-ip appGatewayFrontendIP --frontend-port 8081 --resource-group yr-aks-rg --gateway-name yr-aks-appgateway

az network application-gateway frontend-port create -g yr-aks-rg --gateway-name yr-aks-appgateway --name apiPort --port 8081

az network application-gateway http-listener create -g yr-aks-rg --gateway-name yr-aks-appgateway --name apiListener --frontend-ip appGatewayFrontendIP --frontend-port apiPort

az network application-gateway http-settings create -g yr-aks-rg --gateway-name yr-aks-appgateway --port 3000 --name apiHttpSettings

az network application-gateway rule create -g yr-aks-rg --gateway-name yr-aks-appgateway --rule-type basic --address-pool yr-aks-appgateway-api-pool --http-listener apiListener --name apiRule --http-settings apiHttpSettings

```

You can also setup Path based redirection as indicated here - WIll leave this as an exercise!

You can now confirm that the application is working on the app gateway url by pasting its public IP (without port 8080) to check if the app is working as expected. 
You can also check if the API is now accessible over port 8081. 


## Attribution:
Content originally created by @whereisyusuf et al. from [this](https://github.com/Azure/blackbelt-aks-hackfest/blob/master/labs/day1-labs/10-cluster-upgrading.md) 
