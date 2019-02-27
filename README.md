# Network Policy on AKS
This is a demo to walk through setting up Network Policies on AKS, and their use.  By default, without any Network Policies, any Pods within a cluster namespace are allowed to communicate with each other.  From a security persepctive you can either trust the boundary of the cluster ot implement hard restrictions on workload to workload communications.

## Why!?
Anything running within a container is still an attack vector.  If you are serving a REST endpoint over HTTP, you can still (potentially) inflict a buffer overflow.  Containers don't change this.  They are just a mechanism to Build, Ship and Run your workloads.  If yout B.S.R. bad code it just means it gets shipped quiccker.

### Now what?
With Network Policies you have the ability to limit communication from one workload to another. This allows you to limit the attack vector, and constrain communication to ONLY the workloads that need to talk to each other.  "Every little helps"...

### 

### Yes, but I need you to secure everything
Here's a few tips:

* Scan all code and containers with something like Aqua or Twistlock
* Use Mutual TLS for inter-Pod communications
* Setup namespacaes for Dev/Test/Production
* NEVER run workloads as root (superuser)
* Restrict Container processes to what they need to access (disk, memory limits, CPU limits)

## Enable the preview feature

As this is a preview feature, register it on your account:

```bash
az feature register --name EnableNetworkPolicy --namespace Microsoft.ContainerService
```

This process will take a few minutes to propogate to your subscription, to check the status issue a query to the Azure API:

```bash
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/EnableNetworkPolicy')].{Name:name,State:properties.state}"

Name                                            State
----------------------------------------------  -----------
Microsoft.ContainerService/EnableNetworkPolicy  Registering
```

When the state changes to Registered like this
```bash
Name                                            State
----------------------------------------------  ----------
Microsoft.ContainerService/EnableNetworkPolicy  Registered
```

You need to then run the following to propagate the changes

```bash
az provider register -n Microsoft.ContainerService
```

## Creating your AKS cluster with Network Policy support

Let's go ahead and setup some environment variables we will use later...

```bash
export RESOURCE_GROUP_NAME=juda-AK-NP1
export CLUSTER_NAME=juda-AKS1
```

***Use your Microsoft alias to setup your cluster RG***


### Create a resource group for your new AKS cluster:

```bash
az group create --name $RESOURCE_GROUP_NAME --location eastus
```


### Create a virtual network and subnet
```bash
az network vnet create --resource-group $RESOURCE_GROUP_NAME --name myVnet --address-prefixes 10.0.0.0/8 --subnet-name myAKSSubnet --subnet-prefix 10.240.0.0/16
```

### Get the virtual network resource ID
```bash
VNET_ID=$(az network vnet show --resource-group $RESOURCE_GROUP_NAME --name myVnet --query id -o tsv)
```


### Get the virtual network subnet resource ID
```bash
SUBNET_ID=$(az network vnet subnet show --resource-group $RESOURCE_GROUP_NAME --vnet-name myVnet --name myAKSSubnet --query id -o tsv)
```

### Create the AKS cluster and specify the virtual network and service principal information

```bash
az aks create --resource-group $RESOURCE_GROUP_NAME --name $CLUSTER_NAME --kubernetes-version 1.12.4 --network-plugin azure --service-cidr 10.0.0.0/16 --dns-service-ip 10.0.0.10 --docker-bridge-address 172.17.0.1/16 --vnet-subnet-id $SUBNET_ID --network-policy calico --generate-ssh-keys

```

***Note: You must specify the Kubernetes version to enable Network Policies as it is version specific where the feature has been lit up*** 

It's time for a :coffee: (or :tea:) - this will take a few minutes.


```bash
DnsPrefix                   EnableRbac    Fqdn                                                      KubernetesVersion    Location    Name      NodeResourceGroup              ProvisioningState    ResourceGroup
--------------------------  ------------  --------------------------------------------------------  -------------------  ----------  --------  -----------------------------  -------------------  ---------------
juda-AKS-juda-AK-NP-63bb10  True          juda-aks-juda-ak-np-63bb10-e3e56c90.hcp.eastus.azmk8s.io  1.12.4               eastus      juda-AKS  MC_juda-AK-NP_juda-AKS_eastus  Succeeded            juda-AK-NP
```

### Get AKS credentials

Once you have an AKS cluster up and running, you need to download the Kubernetes context to be able to connect via **kubectl**:

```bash
az aks get-credentials --resource-group $RESOURCE_GROUP_NAME --name $CLUSTER_NAME
```

# What are Network Policies?
Kubernetes is all about the abstraction of workloads from infrastructure.  I don't need to know about virtual machines, I don't care about IP addresses; what I care about is the workload.

In Kubernetes, workloads are label based.  For example, I may have a workload that has the "app=web" label, and another that has "app=db" (to signify a very simple web -> db interaction).

If I independantly scale those, I don't have a list of IP addresses that should or should not talk to each other, as I don't know ahead of time what they will be.  

In traditional IT, where I care for my Pets (disk, network, CPU,  memory), I need to work with the network and security team to limit the communication.  In Kubernetes, I say "workload A talks to workload B and nobody else".

Think of Network Policies as an overlay of my security policy on an IP filter level.  It's kubernetes natice way of enforcing the policy of networking.


# Deploy a Simple API

Let's deploy a simple NGINX Container onto the cluster, we will give it the label "app=api" - this will be important later.  We also expose the Pod 

```bash
kubectl run api --image=nginx --labels app=api --expose --port 80 
```


And not let's test that we can spin up another Container and communicate with the API we just deployed. T

```bash
kubectl run --rm -it --image=alpine test-np
```

Once you have a prompt in the container, try to talk to the API service we just published...

```bash
# wget http://api -qO-

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
````

# Deploy a Deny Policy

Let's deploy a policy to deny all traffic on the Ingress (the Service endpoint).  Apply the [./deny.yaml](deny.yaml) to your cluster.  This will setup a Deny All policy as the ingress component is empty.  This is a deny all, selective allow policy enforcement.

```bash
kubectl create -f [RootDirectory]/deny.yaml
```

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: api-policy
spec:
  podSelector:
    matchLabels:
      app: api
  ingress: []
```

## Retest communications

Re-run the communication test:

Reattach to pod 
```bash
kubectl attach [test-np name] -i -t 
```

```bash
wget -qO- --timeout=2 http://api
```

**Notice we use a timeout here, optherwise you'd be waiting for 30 seconds for the standard wget timeout**


# Deploy the allow policy

Deploy the [./allow.yaml](allow.yaml) policy to the cluster.  This will alow communication between apps with the app=api label:

```bash
kubectl create -f [allow.yaml]
```


```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: api-policy
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app: api
```

## Retest communications
Now that the Network Policy has been updated, you can re-run the client container, but *this time* make sure to label the Pod with "api=api":
List the current pods and display their labels
```bash
kubectl get pods --show-labels 
```

Add label to test-np pod
```bash
kubectl label pod [test-np name] app=api
```

Rerun show labels to check its taken the new label
```bash
kubectl get pods --show-labels 
```

```bash
wget -qO- --timeout=2 http://api
```
