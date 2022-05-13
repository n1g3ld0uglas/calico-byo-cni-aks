# Calico CNI on Azure AKS
You can now run Calico CNI on Azure AKS: <br/>
https://projectcalico.docs.tigera.io/getting-started/kubernetes/managed-public-cloud/aks

## Confirm Calico IPAM is installed
The IPAM plugin can be queried on the default Installation resource.

```
kubectl get installation default -o go-template --template {{.spec.cni.ipam.type}}
```

If your cluster is using Calico IPAM, the above command should return a result of ```Calico```.

## Install Calicoctl
We will follow the official docs for installing the ```calicoctl``` utility: <br/>
https://projectcalico.docs.tigera.io/maintenance/clis/calicoctl/install <br/>
<br/>
Use the following command to download the ```calicoctl``` binary.
```
curl -L https://github.com/projectcalico/calico/releases/download/v3.23.0/calicoctl-linux-amd64 -o calicoctl
```

Set the file to be executable
```
chmod +x ./calicoctl
```

## Migrate from one IP pool to another

Pods are assigned IP addresses from IP pools that you configure in Calico. <br/>
As the number of pods increase, you may need to increase the number of addresses available for pods to use. <br/>

Or, you may need to move pods from a CIDR that was used by mistake. <br/>
Calico lets you migrate from one IP pool to another one on a running cluster without network disruption <br/>
https://projectcalico.docs.tigera.io/networking/migrate-pools <br/>

Let’s run calicoctl with the below command to to see the IP pool, default-ipv4-ippool
```
./calicoctl get ippool -o wide
```

The output should look like this:

```
NAME                  CIDR            NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
default-ipv4-ippool   10.244.0.0/16   true   Never      Always      false      false              all()
```

When we run the below command, we see that a pod is created using the default range (10.244.34.130/32).
```
./calicoctl get wep --all-namespaces
```

```
NAMESPACE          WORKLOAD                                   NODE                                NETWORKS           INTERFACE         
calico-apiserver   calico-apiserver-6b48b9894b-d627h          aks-nodepool1-38669398-vmss000003   10.244.34.130/32   cali14f7d51e2c6   
calico-apiserver   calico-apiserver-6b48b9894b-hwxlx          aks-nodepool1-38669398-vmss000003   10.244.34.129/32   calib74e2fd2d16   
calico-system      calico-kube-controllers-77f96d9656-gd7hm   aks-nodepool1-38669398-vmss000003   10.244.34.133/32   caliea196ffe544   
kube-system        coredns-69c47794-86db7                     aks-nodepool1-38669398-vmss000003   10.244.34.132/32   calie0a5573f330   
kube-system        coredns-69c47794-wmvrj                     aks-nodepool1-38669398-vmss000003   10.244.34.135/32   cali3db72ae2319   
kube-system        coredns-autoscaler-7d56cd888-t4cvr         aks-nodepool1-38669398-vmss000003   10.244.34.134/32   cali15e5361407d   
kube-system        metrics-server-64b66fbbc8-6n2zm            aks-nodepool1-38669398-vmss000003   10.244.34.131/32   cali51656fecf55 
```

Let’s get started changing this pod to the new IP pool (10.0.0.0/16). <br/>
We add a new ```IPPool``` resource with the CIDR range, 10.0.0.0/16.

```
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: new-pool
spec:
  cidr: 10.0.0.0/16
  ipipMode: Always
  natOutgoing: true
```  
