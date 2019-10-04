## Installing Cluster Using Flux in Azure AKS ##

1. Connect to your k8s cluster in azure cli
```bash
az aks get-credentials --resource-group developyourk8s  --name developyourk8s
```
2. Configure Helm in your cluster
```bash
 kubectl -n kube-system create sa tiller

 kubectl create clusterrolebinding tiller-cluster-rule \
 --clusterrole=cluster-admin \
 --serviceaccount=kube-system:tiller

 helm init --skip-refresh --upgrade --service-account tiller
```
3. Add a LoadBalancer to expose your cluster over a public IP
```bash
helm install stable/nginx-ingress --namespace kube-system --name=nginx-ingress

kubectl --namespace kube-system get services -o wide -w nginx-ingress-controller
```
4. Wait until an external IP is assigned to your nginx loadbalancer, then type CTRL+C to free up the console
```bash
IP=$(kubectl describe svc nginx-ingress-controller -n kube-system | grep "LoadBalancer Ingress:   " | cut -d':' -f 2 | tr -d ' ')

echo $IP

DNSNAME="developyourk8s"

PUBLICIPID=$(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[id]" --output tsv)

az network public-ip update --ids $PUBLICIPID --dns-name $DNSNAME

helm install --name cert-manager --namespace kube-system stable/cert-manager
```
5. At this point you should be able to visit http://developyourk8s.eastus.cloudapp.azure.com/ and see "default backend - 404"
6. Now we'll install flux and connect it to our configuration repository
```bash
helm repo add fluxcd https://charts.fluxcd.io

kubectl apply -f https://raw.githubusercontent.com/fluxcd/flux/helm-0.10.1/deploy-helm/flux-helm-release-crd.yaml

helm upgrade -i flux \
--set helmOperator.create=true \
--set helmOperator.createCRD=false \
--set git.url=git@github.com:patrickbadley/developyourk8s-flux-config.git \
--set git.pollInterval="30s" \
--namespace flux \
fluxcd/flux
```
7. Flux generates a ssh key we can use to authorize it to connect to our git repo. Let's retrieve it first (if your result isnt a long string starting with ssh-rsa, try again until you get one)
```bash
kubectl -n flux logs deployment/flux | grep identity.pub | cut -d '"' -f2
```
8. Copy the result
9. Fork this repo (you will need to update references to the repository name in releases/default/developYourK8sRelease.yaml)
10. Add a github deploy key to your new repo  
  a. Under Settings, choose deploy keys  
  b. Click Add   
  c. Name it "flux" and paste the value in the box  
  d. Check the box to allow write access  
  e. Click Add key  
11. Flux will now configure your cluster!
12. One last piece is to configure cert-manager, a tool that manages ssl certificates for us
```bash
helm upgrade cert-manager     stable/cert-manager     --namespace kube-system     --set ingressShim.defaultIssuerName=letsencrypt-prod --set ingressShim.defaultIssuerKind=ClusterIssuer
```
13. Now go to https://developyourk8s.eastus.cloudapp.azure.com/ and see your app running!

### References: ###
1. https://github.com/stefanprodan/gitops-helm
2. https://docs.microsoft.com/en-us/azure/aks/ingress-tls
3. https://blog.n1analytics.com/free-automated-tls-certificates-on-k8s/
4. https://github.com/fluxcd/helm-operator-get-started
5. https://docs.microsoft.com/bs-latn-ba/azure/aks/kubernetes-walkthrough-portal
6. https://github.com/jetstack/cert-manager
7. https://github.com/nginxinc/kubernetes-ingress
