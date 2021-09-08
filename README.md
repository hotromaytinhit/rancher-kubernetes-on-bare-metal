# Deploying Kubernetes on bare metal with Rancher 2.0

## Contents

+ Install Rancher server
+ Create a Kubernetes cluster
+ Add Kubernetes nodes
+ Install StorageOS as the Kubernetes storage class
+ Understand Nginx Ingress in Rancher


### Install Rancher

Create a VM with Docker and Docker Compose installed and install Rancher 2.0 with docker compose:

+ Rancher docker-compose file:
 [docker-compose.yaml](https://github.com/polinchw/rancher-docker-compose/blob/master/docker-compose.yaml)

+ Run these commands to install Rancher with docker compose:
    +  ```git clone https://github.com/hotromaytinhit/rancher-kubernetes-on-bare-metal ```
    +  ```cd rancher-kubernetes-on-bare-metal ```
    +  ```docker-compose up -d```
    +  ```docker logs rancher_server 2>&1 | grep "Bootstrap Password:"```

### Create your Kubernetes cluster with Rancher

Install a custom Kubernetes cluster with Rancher.  Use the 'Custom' cluster.

![Cluster!](images/rancher.png)

### Add Kubernetes nodes and join the Kubernetes cluster

Run the following commands on all the VMs that your Kubernetes cluster will run on.  The final docker command
will have the VM join the new Kubernetes cluster.

Replace the **--server** and **--token** with your Rancher server and cluster token.

```
#!/bin/bash

#sudo apt update
#sudo apt -y dist-upgrade

#Ubuntu (Docker install)
#sudo apt -y install docker.io

sudo apt -y install linux-image-extra-$(uname -r)

#Debian 9 (Docker install)
#sudo apt -y install apt-transport-https ca-certificates curl gnupg2 software-properties-common
#curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
#sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
#sudo apt update
#sudo apt -y install docker-ce

sudo mkdir -p /etc/systemd/system/docker.service.d/
sudo cat <<EOF > /etc/systemd/system/docker.service.d/mount_propagation_flags.conf
[Service]
MountFlags=shared
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker.service

#This is dependent on your Rancher server
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.1.0-rc9 --server https://75.77.159.159 --token rb8k8kkqw55jqnqbbf4ssdjqtw6hndhfxxcghgv8257kx4p6qsqq55 --ca-checksum 641b2888ce3f1091d20149a495d10457154428f440475b42291b6af1b6c0dd06 --etcd --controlplane --worker
```

### Download the kub config file for the cluster

![Helloservice!](images/kube.png)

After you download the kub config file you can use it by running this command:

```
export KUBECONFIG=$HOME/.kube/rancher-config
```

### Install Helm on the cluster

```
git clone https://github.com/polinchw/set-up-tiller

cd set-up-tiller

chmod u+x set-up-tiller.sh

./set-up-tiller.sh

helm init --service-account tiller

```

### Install StorageOS Helm Chart

```
helm repo add storageos https://charts.storageos.com
helm install --name storageos --namespace storageos-operator --version 1.1.3 storageos/storageoscluster-operator
```

## Add the Storage OS Secret
```
apiVersion: v1
kind: Secret
metadata:
  name: storageos-api
  namespace: default
  labels:
    app: storageos
type: kubernetes.io/storageos
data:
  # echo -n '<secret>' | base64
  apiUsername: c3RvcmFnZW9z
  apiPassword: c3RvcmFnZW9z

```

## Add the StorageOSCluster
```
apiVersion: storageos.com/v1
kind: StorageOSCluster
metadata:
  name: example-storageos
  namespace: default
spec:
  secretRefName: storageos-api
  secretRefNamespace: default
  csi:
    enable: true

```


### Set StorageOS as the default storage class

kubectl patch storageclass fast -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

### Using the default Nginx Igress

Rancher automatically installs the nginx ingress controller on all the nodes in the cluster.  
If you are able to expose one of the VMs in the cluster to the outside world with a public IP
then you can connect to the ingress based services on ports 80 and 443.

Any app you want to be accessible through the default nginx ingress must be added to the 'default'
project in Rancher.

