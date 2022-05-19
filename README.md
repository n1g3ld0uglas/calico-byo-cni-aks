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

<br/>
<br/>
<br/>

## Scenario 1: Migrate from one IP pool to another

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
kubectl create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: new-pool
spec:
  cidr: 10.0.0.0/16
  ipipMode: Always
  natOutgoing: true
EOF  
```  

Let’s verify the new IP pool was created correctly. <br/>
It should reference the new CIDR range, ```10.0.0.0/16```.

```
./calicoctl get ippool -o wide
```

```
NAME                  CIDR            NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
default-ipv4-ippool   10.244.0.0/16   true   Never      Always      false      false              all()      
new-pool              10.0.0.0/16     true   Always     Never       false      false              all()
```

Disable the old IP pool - ```default-ipv4-ippool``` <br/>
Firstly, let's list the existing IP pool definition.

```
./calicoctl get ippool -o yaml > pools.yaml
```

```
cat pools.yaml
```

```
apiVersion: projectcalico.org/v3
items:
- apiVersion: projectcalico.org/v3
  kind: IPPool
  metadata:
    creationTimestamp: "2022-05-13T13:01:46Z"
    name: default-ipv4-ippool
    resourceVersion: "2370"
    uid: 5a5d4a43-8993-4390-ba1f-ff7c0dc45c5f
  spec:
    allowedUses:
    - Workload
    - Tunnel
    blockSize: 26
    cidr: 10.244.0.0/16
    ipipMode: Never
    natOutgoing: true
    nodeSelector: all()
    vxlanMode: Always
- apiVersion: projectcalico.org/v3
  kind: IPPool
  metadata:
    creationTimestamp: "2022-05-13T15:36:40Z"
    managedFields:
    - apiVersion: projectcalico.org/v3
      fieldsType: FieldsV1
      fieldsV1:
        f:spec:
          f:cidr: {}
          f:ipipMode: {}
          f:natOutgoing: {}
      manager: kubectl-create
      operation: Update
      time: "2022-05-13T15:36:40Z"
    name: new-pool
    resourceVersion: "19819"
    uid: 3fa257a0-9c6d-4394-95c7-22e7176717b4
  spec:
    allowedUses:
    - Workload
    - Tunnel
    blockSize: 26
    cidr: 10.0.0.0/16
    ipipMode: Always
    natOutgoing: true
    nodeSelector: all()
    vxlanMode: Never
kind: IPPoolList
metadata:
  resourceVersion: "20996"
```

Disable this IP pool by setting: ```disabled: true``` <br/>
Add this one line after the ```natOutgoing: true``` field.
```
vi pools.yaml
```

Apply the changes. <br/>
Remember, disabling a pool only affects new IP allocations; networking for existing pods is not affected.

```
./calicoctl apply -f pools.yaml
```

Output should look similar to the below example. <br/>
Verify the changes with the same ```./calicoctl get``` command

```
Successfully applied 2 'IPPool' resource(s)
```

```
./calicoctl get ippool -o wide
```

```
NAME                  CIDR            NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
default-ipv4-ippool   10.244.0.0/16   true   Never      Always      true       false              all()      
new-pool              10.0.0.0/16     true   Always     Never       false      false              all()
```

Next, we delete all of the existing pods from the old IP pool. <br/>
If you have multiple pods, you would trigger a deletion for all pods in the cluster.

```
kubectl delete pod -n kube-system <example-coredns-pod>
```

Verify that new pods get an address from the new IP pool. <br/>
Create a test namespace and nginx pod to go in that namespace.

```
kubectl create ns ippool-test
```

```
kubectl -n ippool-test create deployment nginx --image nginx
```

Verify that the new pod gets an IP address from the new range. <br/>
Once you have completed this, cleanup the ```ippool-test``` namespace.

```
kubectl -n ippool-test get pods -l app=nginx -o wide
```

```
kubectl delete ns ippool-test
```

Now that you’ve verified that pods are getting IPs from the new range, you can safely delete the old pool. <br/>
We can now proceed to our next configuration test.

```
./calicoctl delete pool default-ipv4-ippool
```

<br/>
<br/>
<br/>

## Scenario 2: Change IP pool block size
https://projectcalico.docs.tigera.io/networking/change-block-size <br/>
By default, Calico uses an IPAM block size of 64 addresses – /26 for IPv4, and /122 for IPv6. <br/>
However, the block size can be changed depending on the IP pool address family. <br/>
<br/>
IPv4: 20-32, inclusive
IPv6: 116-128, inclusive
You can have only one default IP pool for per protocol in your installation manifest. <br/>
In this example, there is one IP pool for IPv4 (/26), and one IP pool for IPv6 (/122) <br/>

```
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
  pec:
   # Configures Calico networking.
   calicoNetwork:
     # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.48.0.0/21
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
    - blockSize: 122
      cidr: 2001::00/64 
      encapsulation: None 
      natOutgoing: Enabled 
      nodeSelector: all()
```

However, the following is ```INVALID``` because it has two IP pools for IPv4.
```
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
  spec:
   # Configures Calico networking.
   calicoNetwork:
     # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.48.0.0/21
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
    - blockSize: 31
      cidr: 10.48.8.0/21
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
```

#### Expand or shrink IP pool block sizes
By default, the Calico IPAM block size for an IP pool is /26. <br/>
To expand from the default size /26, lower the blockSize (for example, /24). <br/>
To shrink the blockSize from the default /26, raise the number (for example, /28).


#### Best practice: change IP pool block size before installation
We now know the ```blockSize``` field cannot be edited directly after Calico installation. <br/>
It is best to change the IP pool block size before installation to minimize disruptions to pod connectivity.

#### Create a temporary IP pool
We add a new IPPool with the CIDR range, ```10.0.0.0/16```.

```
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: temporary-pool
spec:
  cidr: 10.0.0.0/16
  ipipMode: Always
  natOutgoing: true
```

Apply the changes.
```
./calicoctl apply -f temporary-pool.yaml
```

Let’s verify the temporary IP pool.
```
./calicoctl get ippool -o wide
```
```
NAME                  CIDR             NAT    IPIPMODE   DISABLED
default-ipv4-ippool   192.168.0.0/16   true   Always     false
temporary-pool        10.0.0.0/16      true   Always     false
```

#### Disable the existing IP pool
Disable allocations in the default pool.
```
./calicoctl patch ippool default-ipv4-ippool -p '{"spec": {"disabled": true}}'
```

Verify the changes.
```
./calicoctl get ippool -o wide
```
```
NAME                  CIDR             NAT    IPIPMODE   DISABLED
default-ipv4-ippool   192.168.0.0/16   true   Always     true
temporary-pool        10.0.0.0/16      true   Always     false
```

## Delete pods from the existing IP pool
In our example, coredns is our test pod scenario <br/> 
For multiple pods you would trigger a deletion for all pods in the cluster.
```
kubectl delete pod -n kube-system coredns-6f4fd4bdf-8q7zp
```

Restart all pods with just one command. <br/>
```WARNING!``` The following command is disruptive and may take several minutes depending on the number of pods deployed.

```
kubectl delete pod -A --all
```

#### Delete the existing IP pool
Now that you’ve verified that pods are getting IPs from the new range, you can safely delete the existing pool.
```
./calicoctl delete ippool default-ipv4-ippool
```





<br/>
<br/>
<br/>


## Calico Certified Courses
For more configuration scenarios, users can sign-up for the certified Calico Operator (for Azure) course:
```
[https://academy.tigera.io/course/certified-calico-operator-level-1/](https://www.tigera.io/lp/calico-academy-completion-azure/)
```

<br/>
<br/>
<br/>

## Connecting to Calico Cloud

Although not mention in our docs, the install process worked for Calico Cloud <br/>
https://docs.calicocloud.io/get-started/connect/aks <br/>

<img width="1000" alt="Screenshot 2022-05-13 at 17 20 08" src="https://user-images.githubusercontent.com/82048393/168325678-27e884cd-6e53-4e12-984c-c069bb1e6ebc.png">

Check the installation status with the below command:
```
kubectl get installer default --namespace calico-cloud -o jsonpath --template '{.status}'
```

At this point, I recommend using our supported checklist for troubleshooting advice: <br/>
https://docs.calicocloud.io/get-started/connect/checklist

```
{"clusterName":"my-new-cluster","state":"installing"}
```
After state is ```complete```, Calico Cloud is properly installed. <br/>
<br/>

The ```Network Profile``` will always show as ```CNI=None``` - even after installing Calico CNI plugin
```
az aks show --resource-group my-calico-rg --name my-calico-cluster --query 'networkProfile'
```

<img width="988" alt="Screenshot 2022-05-13 at 17 25 27" src="https://user-images.githubusercontent.com/82048393/168326599-dfc7f1a7-6d83-4cf0-89e4-c4e66bc5761d.png">

Check the status of all Calico Cloud components via the below command <br/>
NB: Wait until all components are set to ```AVAILABLE: true```:

```
kubectl get tigerastatus -w
```

<img width="1004" alt="Screenshot 2022-05-13 at 20 10 01" src="https://user-images.githubusercontent.com/82048393/168374099-246e6623-44e7-42ec-8d30-53226dc86136.png">




