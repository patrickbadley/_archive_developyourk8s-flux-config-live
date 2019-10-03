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
5. At this point you should be able to visit http://developyourk8s.eastus.cloudapp.azure.com/ and see "no backend"
6. Now we'll install flux and connect it to our configuration repository
```bash
IP=$(kubectl describe svc nginx-ingress-controller -n kube-system | grep "LoadBalancer Ingress:   " | cut -d':' -f 2 | tr -d ' ')
```

### References: ###
1. https://github.com/stefanprodan/gitops-helm
2. https://docs.microsoft.com/en-us/azure/aks/ingress-tls
3. https://blog.n1analytics.com/free-automated-tls-certificates-on-k8s/
