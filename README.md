## Installing reference architecture in Azure AKS ##

0. AKS Setup Notes:  
  a. Enable RBAC  
  b. Disable HTTP application routing add-on (this is the default now)
1. Create a service account for tiller (helm server side component)
```bash
kubectl -n kube-system create sa tiller
```
2. Give admin rights to tiller
```bash
kubectl create clusterrolebinding tiller-cluster-rule \
	--clusterrole=cluster-admin \
	--serviceaccount=kube-system:tiller 
```
3. Initialize helm for you cluster
```bash
helm init --skip-refresh --upgrade --service-account tiller
```
4. Install nginx ingress controller
```bash
helm install stable/nginx-ingress --namespace kube-system --name=nginx-ingress
```
5. Get ip address of ingress controller: (under External Ip Column)
```bash
IP=$(kubectl describe svc nginx-ingress-controller -n kube-system | grep "LoadBalancer Ingress:   " | cut -d':' -f 2 | tr -d ' ')
```
6. Verify the Public IP address of your ingress controller was ready (if it says <pending> run the previous command again)
```bash
echo $IP
```
7. Save a friendly host name for your cluster components
```bash
DNSNAME="hmb-dev-ops-app-dev"
```
8. Get the azure resource-id of the public ip
```bash
PUBLICIPID=$(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[id]" --output tsv)
```
9. Update public ip address with DNS name (will result in a hostname like: bmw-ref-arch.eastus.cloudapp.azure.com)
```bash
az network public-ip update --ids $PUBLICIPID --dns-name $DNSNAME
```
10. Navigate to http://bmw-ref-arch-simple.eastus.cloudapp.azure.com/ and you should see "default backend - 404"
11. Install Cert-Manager
```bash
helm install --name cert-manager --namespace kube-system stable/cert-manager
```
12. Fork github.com/patrickbadley/devops-team-flux-config and update any references to your images/repositories/servername
13. (optional) If using a private container registry: Get credentials to your azure container registry
```bash
ACR_NAME=hmbdevopsteam
SERVICE_PRINCIPAL_NAME=acr-service-principal
ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --query loginServer --output tsv)
ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --query id --output tsv)
SP_PASSWD=$(az ad sp create-for-rbac --name $SERVICE_PRINCIPAL_NAME --role Reader --scopes $ACR_REGISTRY_ID --query password --output tsv)
CLIENT_ID=$(az ad sp show --id http://$SERVICE_PRINCIPAL_NAME --query appId --output tsv)
kubectl create secret docker-registry azure-reg-creds --docker-server $ACR_NAME.azurecr.io --docker-username $CLIENT_ID --docker-password $SP_PASSWD --docker-email pjb2@hmbnet.com
```
14. Add flux helm chart repository
```bash
helm repo add weaveworks https://weaveworks.github.io/flux
```
15. Install flux using helm and passing in your config repository url (fork mine and update the image locations as needed first)
```bash
helm install --name flux --set rbac.create=true --set helmOperator.create=true --set git.url=ssh://git@github.com/patrickbadley/devops-team-flux-config  --namespace flux weaveworks/flux
```
16. Get flux's ssh key that it will use to authenticate with your git repository
```bash
kubectl -n flux logs deployment/flux | grep identity.pub | cut -d '"' -f2
```
17. Add the printed key to your git repositories Deploy keys with Write access
18. Monitor the helm operator pod, all your pods, and helm releases until they get spun up and then navigate to the url above!
```bash
kubectl logs --selector=app=flux-helm-operator -n flux
```
```bash
kubectl get pods --all-namespaces
```
```bash
helm list
```
19. Update your Cert-manager installation with the cluster issuer created by your flux release:
```bash
helm upgrade cert-manager     stable/cert-manager     --namespace kube-system     --set ingressShim.defaultIssuerName=letsencrypt-prod --set ingressShim.defaultIssuerKind=ClusterIssuer
```

### References: ###
1. https://github.com/stefanprodan/gitops-helm
2. https://docs.microsoft.com/en-us/azure/aks/ingress-tls
3. https://blog.n1analytics.com/free-automated-tls-certificates-on-k8s/
4. https://netflix.github.io/conductor/
5. https://istio.io/docs/

### Source Code for Apps ###
1. https://github.com/patrickbadley/ref-arch-sql

#### Notes: ####
1. To secure with let's encrypt follow the instructions below to set up a dns01 provider (I haven't tried this yet)  
  a. https://github.com/jetstack/cert-manager/issues/591  
  b. https://medium.com/@prune998/istio-0-8-0-envoy-cert-manager-lets-encrypt-for-tls-d26bee634541
