# Cluster API on AZURE <!-- omit in toc -->

### Working with **release-0.3** providing support for **v1alpha2** <!-- omit in toc -->

#### Table of Content <!-- omit in toc -->
- [- 3 nodes in this example.](#ulli3-nodes-in-this-exampleliul)
    - [Generating the Kubeconfig file](#generating-the-kubeconfig-file)
    - [Install the CNI addons to the Workload Cluster.](#install-the-cni-addons-to-the-workload-cluster)
    - [Deploy the Worker Nodes leveraging the Cluster API.](#deploy-the-worker-nodes-leveraging-the-cluster-api)
  - [Lifecycle management of Workload Cluster](#lifecycle-management-of-workload-cluster)
  - [References](#references)


-----
## Setup 
This guide will use an upto date Ubuntu 18.04 server to run a management cluster (the  management cluster can be run on any K8s cluster (pivoting)). For simplicity sake, we will be using a simple K8s cluster, deployed using `kind`. Once the management cluster is setup, it can be used to install and manage workload clusters.

### Preparations

Make sure the following packages are installed on the Ubuntu server and are up to date. 

* JQ
* golang v1.13.4 at the time of writing this doc.
```shell
sudo add-apt-repository ppa:longsleep/golang-backports
sudo apt-get update
sudo apt install golang
```
* git
* curl
* kubectl
* kind
```shell
curl -Lo ./kind https://github.com/kubernetes-sigs/kind/releases/download/v0.7.0/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin
```
* azure cli [installed](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-apt?view=azure-cli-latest) and [configured/loggedi in](https://docs.microsoft.com/en-us/cli/azure/get-started-with-azure-cli?view=azure-cli-latest) to access your Azure account. Click on the links for additional directions. 
* Copy the `SubscriptionID` of the account that will be used - 
```console
az account list -o table
```
* Create a new Azure Active Directory service principals for automation authentication or use an existing one -
```console
az ad sp create-for-rbac -n "SPClusterAPI"
```

* Export the following environment variables. These variables ***should be modified as per user's requirements***. 

```console
export AZURE_SUBSCRIPTION_ID="YOUR_SUBSCRIPTION_ID"
export AZURE_TENANT_ID="YOUR_TENENT_ID"
export AZURE_CLIENT_ID="YOUR_CLIENT_ID"
export AZURE_CLIENT_SECRET="YOUR_CLIENT_SECRET"
export AZURE_LOCATION="eastus"
export CLUSTER_NAME=workload-cluster-az-1
export KUBERNETES_VERSION=1.16.3
```

### Clone the stable `release-0.3` branch that provides `v1alpha2` support 
```shell
git clone https://github.com/kubernetes-sigs/cluster-api-provider-azure.git  --branch release-0.3
cd cluster-api-provider-azure
```

## Deploying the Management Cluster 

Generate the sample yaml files. These files are required to setup the management cluster (CRDs/controllers) and are also needed to setup the workload clusters. 

```shell
make generate-examples
```
All the yamls are generated in the `./example/_out` folder

Create a simple management cluster using `kind`

```shell
kind create cluster --name=clusterapi
``` 
Within a few minutes, the management cluster is instantiated in the Ubuntu server!!!

```console
Creating cluster "clusterapi" ...
 ✓ Ensuring node image (kindest/node:v1.17.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-clusterapi"
You can now use your cluster with:

kubectl cluster-info --context kind-clusterapi

Thanks for using kind! 😊
```
Make sure the cluster is accessible using the kubeconfig file.

```shell
kubectl get pods --all-namespaces
```
should return something similar (with all PODS in a `Running` state)

```console
NAMESPACE            NAME                                               READY   STATUS    RESTARTS   AGE
kube-system          coredns-6955765f44-92gc8                           1/1     Running   0          104s
kube-system          coredns-6955765f44-snqhv                           1/1     Running   0          104s
...
kube-system          kube-proxy-5mqvv                                   1/1     Running   0          104s
kube-system          kube-scheduler-clusterapi-control-plane            1/1     Running   0          2m
local-path-storage   local-path-provisioner-7745554f7f-ktlj5            1/1     Running   0          104s
```

Using the following 3 commands, install the Cluster API CRDs and controllers to this newly created management cluster -

```shell
kubectl apply -f https://github.com/kubernetes-sigs/cluster-api/releases/download/v0.2.9/cluster-api-components.yaml
kubectl create -f https://github.com/kubernetes-sigs/cluster-api-bootstrap-provider-kubeadm/releases/download/v0.1.5/bootstrap-components.yaml

# Create the base64 encoded credentials
export AZURE_SUBSCRIPTION_ID_B64="$(echo -n "$AZURE_SUBSCRIPTION_ID" | base64 | tr -d '\n')"
export AZURE_TENANT_ID_B64="$(echo -n "$AZURE_TENANT_ID" | base64 | tr -d '\n')"
export AZURE_CLIENT_ID_B64="$(echo -n "$AZURE_CLIENT_ID" | base64 | tr -d '\n')"
export AZURE_CLIENT_SECRET_B64="$(echo -n "$AZURE_CLIENT_SECRET" | base64 | tr -d '\n')"
curl -L https://github.com/kubernetes-sigs/cluster-api-provider-azure/releases/download/v0.3.0/infrastructure-components.yaml \
  | envsubst \
  | kubectl create -f -

```
This should create all the necessary CRDs and controllers in the cluster, with an output similar to this -
```console
namespace/cabpk-system created
namespace/capz-system created
namespace/capi-system created
...
customresourcedefinition.apiextensions.k8s.io/clusters.cluster.x-k8s.io created
...
secret/capz-manager-bootstrap-credentials created
service/cabpk-controller-manager-metrics-service created
service/capz-controller-manager-metrics-service created
deployment.apps/cabpk-controller-manager created
deployment.apps/capz-controller-manager created
deployment.apps/capi-controller-manager created
```
Make sure that all the new controllers are in a running state -

```shell
kubectl get pods --all-namespaces
```

should return similar to below. The `capi-system`,`capz-system` and `cabpk-system` are the new namespaces with the controllers running in them - 

```console
NAMESPACE            NAME                                               READY   STATUS    RESTARTS   AGE
cabpk-system         cabpk-controller-manager-c58d8596f-7vmkx                 2/2     Running   0          3d5h
capi-system          capi-controller-manager-6c64c695bb-qcmmq                 1/1     Running   0          3d5h
capz-system          capz-controller-manager-5f95547946-mb667                 1/1     Running   0          3d4h
...
```

With this, the management cluster is now ready. You can now start deploying workload-clusters by modifying/tweaking the other yaml files in the `./example/_out` folder. 

----
## Deploying the Workload Cluster(s) 

These steps would need to be performed for each Workload cluster that need to be deployed. Obviously, settings and variables need to be adjusted accordingly. 

### Creating the Cluster object

Note - If you want to have use your custom VPC CIDR block, you can modify the `cluster.yaml` accordingly - 

```yaml
...
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
kind: AzureCluster
metadata:
  name: workload-cluster-az-1
  namespace: default
spec:
  location: eastus
# Modify this section with custom cidr blocks and as needed.
#--------------------------------
  networkSpec:
    subnets:
    - name: ControlPlane
      cidrBlock: 10.20.0.0/24
      role: ControlPlane
    - name: Node
      cidrBlock: 10.20.1.0/24
      role: Node
    vnet:
      name: workload-cluster-vnet
      cidrBlock: 10.20.0.0/16
  resourceGroup: workload-cluster-az-1
 ---
 ...
```

Now create the cluster object.

```shell
kubectl apply -f ./examples/_out/cluster.yaml
```
Should display an output similar to this - 
```console
cluster.cluster.x-k8s.io/workload-cluster-az-1 created
azurecluster.infrastructure.cluster.x-k8s.io/workload-cluster-az-1 created
```
Validate within the Azure console that a new resourcegroup with associated subnets, security groups, LB, and other objects has been created. This should take approx. 5 mins. 

### Creating the Control Plane machines. 
- 3 nodes in this example. 
=============================================
```shell
kubectl apply -f ./examples/_out/controlplane.yaml
```
Should display an output similar to this -
```console
kubeadmconfig.bootstrap.cluster.x-k8s.io/workload-cluster-az-1-controlplane-0 created
kubeadmconfig.bootstrap.cluster.x-k8s.io/workload-cluster-az-1-controlplane-1 created
kubeadmconfig.bootstrap.cluster.x-k8s.io/workload-cluster-az-1-controlplane-2 created
machine.cluster.x-k8s.io/workload-cluster-az-1-controlplane-0 created
machine.cluster.x-k8s.io/workload-cluster-az-1-controlplane-1 created
machine.cluster.x-k8s.io/workload-cluster-az-1-controlplane-2 created
azuremachine.infrastructure.cluster.x-k8s.io/workload-cluster-az-1-controlplane-0 created
azuremachine.infrastructure.cluster.x-k8s.io/workload-cluster-az-1-controlplane-1 created
azuremachine.infrastructure.cluster.x-k8s.io/workload-cluster-az-1-controlplane-2 created
```

This may take a while - 10-15 mins. 

### Generating the Kubeconfig file

Once this is complete, we need to grab the kubeconfig file that was associated with this workload server. The `kubeconfig` is stored as a secret within the management cluster, in the namespace associated with the workload cluster (default in this example) . This will allow us to connect to the workload cluster. 

```shell
kubectl get secrets -n default workload-cluster-kubeconfig -o json |jq -r .data.value|base64 -d > /tmp/workload-cluster.conf
```
The `workload-cluster.conf` file should be in a similar kubeconfig file format -

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5ekNDQWJPZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01ERXhPREF6TXpBd01Gb1hEVE13TURFeE5UQXpNelV3TUZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTFN5CjhLYy9lTVdCMzN2MXpaWVpkNjdubE8rd1oyUEN1Z0dIeXE0dTU3eDJ3YmJFcy9mRGpMVWZQOUJ1R1M5Ym4wY3QKeWthZVRrcGpyNk96dzRyR0E0dm1ZVUtLbFNkbXp...
    ...
contexts:
- context:
    cluster: workload-cluster
...
```
###  Install the CNI addons to the Workload Cluster.

```shell
kubectl --kubeconfig=/tmp/workload-cluster.conf apply -f ./examples/addons.yaml
```

This should deploy the required addon objects in the workload cluster -

```console
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
...
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
```

Validate all the pods are successfully running. 

```shell
kubectl --kubeconfig=/tmp/workload-cluster.conf get pods --all-namespaces
```
Should display an output similar to this -

```console
NAMESPACE     NAME                                                               READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-564b6667d7-mbw5f                           1/1     Running   0          102s
kube-system   calico-node-8plqr                                                  1/1     Running   0          102s
kube-system   calico-node-gl8vf                                                  1/1     Running   0          102s
kube-system   calico-node-rjdvd                                                  1/1     Running   0          102s
kube-system   coredns-5644d7b6d9-jtdj4                                           1/1     Running   0          10m
kube-system   coredns-5644d7b6d9-r9mlv                                           1/1     Running   0          10m
kube-system   etcd-ip-10-0-0-18.us-east-2.compute.internal                       1/1     Running   0          9m43s
...
```
###  Deploy the Worker Nodes leveraging the Cluster API.

```shell
kubectl apply -f ./examples/_out/machinedeployment.yaml
```
Should display an output similar to this -

```console
kubeadmconfigtemplate.bootstrap.cluster.x-k8s.io/workload-cluster-az-1-md-0 created
machinedeployment.cluster.x-k8s.io/workload-cluster-az-1-md-0 created
azuremachinetemplate.infrastructure.cluster.x-k8s.io/workload-cluster-az-1-md-0 created
```
In a short time, a machine deployment consisting of 2 machines (2 worker nodes) will be added to the workload cluster. 

```shell
kubectl --kubeconfig=/tmp/workload-cluster.conf get nodes
```
Should display an output similar to this -

```console
NAME                                   STATUS   ROLES    AGE     VERSION
workload-cluster-az-1-controlplane-0   Ready    master   11m     v1.16.3
workload-cluster-az-1-controlplane-1   Ready    master   9m28s   v1.16.3
workload-cluster-az-1-controlplane-2   Ready    master   9m17s   v1.16.3
workload-cluster-az-1-md-0-gldsn       Ready    <none>   107s    v1.16.3
workload-cluster-az-1-md-0-h4htn       Ready    <none>   111s    v1.16.3
```
## Lifecycle management of Workload Cluster

Please use the LCM of Workload Clusters for AWS to understand similar options available for Azure. Note that the v2 of Azure Cluster API does not deploy a bastion host (as required by the specifications). A new bastion would need to be installed in the VNET to enable access to the Control plane nodes and Worker nodes. 


## References

1. [https://blog.scottlowe.org/2019/08/27/bootstrapping-a-kubernetes-cluster-on-aws-with-clusterapi/](https://blog.scottlowe.org/2019/08/27/bootstrapping-a-kubernetes-cluster-on-aws-with-clusterapi/)
2. [https://cluster-api.sigs.k8s.io/tasks/installation.html](https://cluster-api.sigs.k8s.io/tasks/installation.html)
3. [https://github.com/kubernetes-sigs/cluster-api-provider-aws/blob/v0.3.7/docs/getting-started.md](https://github.com/kubernetes-sigs/cluster-api-provider-aws/blob/v0.3.7/docs/getting-started.md)
4. [https://blog.chernand.io/2019/03/19/getting-familiar-with-clusterapi/](https://blog.chernand.io/2019/03/19/getting-familiar-with-clusterapi/)
5. [https://medium.com/condenastengineering/clusterapi-a-guide-on-how-to-get-started-ff9a81262945](https://medium.com/condenastengineering/clusterapi-a-guide-on-how-to-get-started-ff9a81262945).
