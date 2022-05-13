# calico-byo-cni-aks


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
