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
4. Download Istio: (more info at: https://istio.io/docs/setup/kubernetes/download-release/)
```bash
curl -L https://git.io/getLatestIstio | sh -
cd istio-1.0.4
```
5. Install Istio
```bash
helm install install/kubernetes/helm/istio --name istio --namespace istio-system \
	--set grafana.enabled=true \
	--set servicegraph.enabled=true \
	--set tracing.enabled=true \
	--set global.mtls.enabled=true
```
5. Get ip address of ingress gateway: (under External Ip Column)
```bash
IP=$(kubectl describe svc istio-ingressgateway -n istio-system | grep "LoadBalancer Ingress:   " | cut -d':' -f 2 | tr -d ' ')
```
6. Verify the Public IP address of your ingress controller was ready (if it says <pending> run the previous command again)
```bash
echo $IP
```
7. Save a friendly host name for your cluster components
```bash
DNSNAME="bmw-ref-arch"
```
8. Get the azure resource-id of the public ip
```bash
PUBLICIPID=$(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[id]" --output tsv)
```
9. Update public ip address with DNS name (will result in a hostname like: bmw-ref-arch.eastus.cloudapp.azure.com)
```bash
az network public-ip update --ids $PUBLICIPID --dns-name $DNSNAME
```
10. Generate a TLS Certificate for the fully qualified hostname you created above and store the key and value in a kubernetes secret (name and namespace below are important):
```yaml
apiVersion: v1
data:
  tls.crt: <insert your tls certificate here>
  tls.key: <insert your tls certificate key here>
kind: Secret
metadata:
  name: istio-ingressgateway-certs
  namespace: istio-system
type: kubernetes.io/tls
```
11. Fork github.com/patrickbadley/microservice-reference-architecture-config-istio and update the following files:  
  a. \charts\ingress-management\values.yaml (domainName and contactEmail)  
  b. \releases\\{dev/stg/prod}\order-workflow\values.yaml (image dockerhub repositories)  
  c. \releases\\{dev/stg/prod}\values.yaml (image dockerhub repositories)
12. Add flux helm chart repository
```bash
helm repo add weaveworks https://weaveworks.github.io/flux
```
13. Install flux using helm and passing in your config repository url (fork mine and update the image locations as needed first)
```bash
helm install --name flux --set rbac.create=true --set helmOperator.create=true --set git.url=ssh://git@github.com/patrickbadley/microservice-reference-architecture-config-istio --set git.pollInterval=1m --namespace flux weaveworks/flux
```
14. Get flux's ssh key that it will use to authenticate with your git repository
```bash
kubectl -n flux logs deployment/flux | grep identity.pub | cut -d '"' -f2
```
15. Add the printed key to your git repositories Deploy keys with Write access
16. Monitor the helm operator pod, all your pods, and helm releases until they get spun up and then navigate to the url above!
```bash
kubectl logs --selector=app=flux-helm-operator -n flux
```
```bash
kubectl get pods --all-namespaces
```
```bash
helm list
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
