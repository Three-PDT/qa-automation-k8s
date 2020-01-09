# QA Kubernetes cluster deployment

### Technologies needed
kubernetes-cli
kubernetes installed(v1.15.7 used here) (see upgrading kubernetes for reference)
azure cli installed
helm installed(v3 used here)

### CLUSTER SETUP:
1. Login  
	`az login`
2. Set active subscription (CIO - Digital)  
	`az account set`  
3. Create a resource group (Can be done via portal, might need to switch local cluster context)  
	`az group create --name QaAutomationResources --location westeurope`
4. Create a named managed cluster  
	`az aks create -g QaAutomationResources -n QaTestAutomation --node-count 3 --enable-addons monitoring --location westeurope  --kubernetes-version 1.15.7 > create-response.json`  
5. Make sure your cluster context is set to the newly created cluster. --overwrite-existing flag necessary if you're redeploying a cluster  
	`az aks get-credentials -g QaAutomationResources -n QaTestAutomation --admin --overwrite-existing`  
6. Enable the k8s dashboard for the cluster  
	`kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard`

### HELM SETUP:

1. Initialise  
	`helm init`
2. Add the helm charts via "stable" repo  
	`helm repo add stable https://kubernetes-charts.storage.googleapis.com/`

### INGRESS:

Create ingress namespace

1. kubectl create namespace ingress-basic
2. Install an nginx-ingress controller

```bash
helm install test-automation-ingress stable/nginx-ingress \
    --namespace ingress-basic \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
```


### Setting up DNS:

Find the public IP of the ingress controller.  
Do this by going to the dashboard, making sure the namspace is set to 'All namespaces', and going to Services in the menu.  
From there, you can find the public IP from the ingress-controller in the 'External endpoints' column.
Now run the following command, replacing `%YOUR_IP%` with the one above:
Run:

```bash
az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '%YOUR_IP%')].[id]" --output tsv
```

The result is a Uri for the public Ip details within Azure.  
Replace `%YOUR_IP_URI%` and `%YOUR_DNS%` (I tend to use the name of the AKS cluster for this) in the following:
Run:

```bash
az network public-ip update --ids "%YOUR_IP_URI%" --dns-name "%YOUR_DNS%" > create-response.json
```

NOTE!
regression-qa is the dns used for test automation. jenkins.three-digital.co.uk is c-named to regression-qa.westeurope.cloudapp.azure.com

Example input:   
1.

```
az network public-ip list --query "[?ipAddress!=null]|
[?contains(ipAddress, '51.137.55.188')].[id]" --output tsv
```

EXAMPLE IP URI OUTPUT: 

```bash
/subscriptions/a5b4936a-c991-4986-a086-669067406f2b/resourceGroups/
mc_qaautomationresources_qatestautomation_westeurope/providers/
Microsoft.Network/publicIPAddresses/kubernetes-a1a26003156754aeca3d1f6d6fe879d0
```

2.

```bash
az network public-ip update --ids "/subscriptions/a5b4936a-c991-4986-a086-669067406f2b/resourceGroups/
mc_qaautomationresources_qatestautomation_westeurope/providers/
Microsoft.Network/publicIPAddresses/kubernetes-a1a26003156754aeca3d1f6d6fe879d0"
 --dns-name "regression-qa" > create-response.json
```

### DEPLOY CERT MANAGER:

https://cert-manager.io/docs/installation/kubernetes/

1. `kubectl create namespace cert-manager`
2. `kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.12.0/cert-manager.yaml`

Verify installation:  
[Follow the cert-manager guide for this](https://cert-manager.io/docs/installation/kubernetes/#verifying-the-installation)

1. verify pods running  
	`kubectl get pods --namespace cert-manager`  
	(3 pods running)
2. Make a test issuer  
	`kubectl apply -f cert-manager/test-issuer.yaml`
3. Check the status of the newly created certificate. You may need to wait a few seconds before cert-manager processes the certificate request.  
	`kubectl describe certificate -n cert-manager-test`  
	Reason will say cert-issued
4. delete test resource  
	`kubectl delete -f cert-manager/test-issuer.yaml`

##### CLUSTER CERT ISSUER:
##### STAGING FIRST:

1. `kubectl apply -f cert-manager/cluster-issuer-staging.yaml`
2. `kubectl apply -f ingress-routes/staging-routes.yaml`

Once the routes are all verified working (after the cert manager), then delete and add production

Delete:  

1. `kubectl delete -f cert-manager/cluster-issuer-staging.yaml`
2. `kubectl delete -f ingress-routes/staging-routes.yaml`

#### PRODUCTION:

1. `kubectl apply -f cert-manager/cluster-issuer-prod.yaml`
2. `kubectl apply -f ingress-routes/production-routes.yaml`

CREATE A PERSISTENT VOLUME AND PERSISTENT VOLUME CLAIM

This is for Jenkins. Jenkins will use an already set up persistent volume and claim.

1. `kubectl apply -f persistent-volume/pv-volume.yaml`
2. `kubectl apply -f persistent-volume/pv-claim.yaml`


#### DEPLOY JENKINS:

1. install jenkins with helm
	`helm install jenkins-master stable/jenkins -f ./jenkins-helm/jenkins-regression.yaml`

NOTE: Command to get the admin password that tends to get lost amongst the output:  
	
```
printf $(kubectl get secret --namespace default \
jenkins-master -o jsonpath="{.data.jenkins-admin-password}" | \
base64 --decode);echo
```
