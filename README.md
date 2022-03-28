# Deploying-Openwhisk-on-Kubernetes

### **Install Docker**
---
_Note: Repeat on all the nodes_
```
dpkg -l containerd
sudo apt install containerd
sudo apt-get update
sudo apt-get install -y docker.io
```

* Set Docker to launch at boot by entering the following:

``` sudo systemctl enable docker```

* Verify Docker is running:

``` sudo systemctl status docker ```

* To start Docker if it’s not running:
``` sudo systemctl start docker ``

* To restart Docker
``` sudo systemctl restart docker```

* Command to Uninstall Docker:

``` sudo apt-get remove docker docker-engine docker.io containerd runc```

### **Install Kubernetes and kubeadm**
---
_Note: Repeat on all the nodes_
```
sudo su root
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list deb http://apt.kubernetes.io/ kubernetes-xenial main EOF
exit
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubernetes-cni
```

* Commands to uninstall Kubernetes:
``` 
sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube*   
sudo apt-get autoremove  
sudo rm -rf ~/.kube
```

* To reser kubedam

```sudo kubeadm reset```

* To restart kubelet

```sudo systemctl restart kubelet```

### Begin Kubernetes Deployment
---
* Start by disabling the swap memory on each server:

```sudo swapoff -a```

* Assign Unique Hostname for Each Server Node 
    * Decide which server to set as the master node. Then enter the command:
     
     ```sudo hostnamectl set-hostname master-node ```
     
    * Next, set a worker node hostname by entering the following on the worker server:
     
     ```sudo hostnamectl set-hostname worker01```
     
  If you have additional worker nodes, use this process to set a unique hostname on each.
  
*  Stop and Disable firewall
```
sudo systemctl stop firewalld
// Check status
sudo firewall-cmd --state
sudo systemctl status firewalld
```
#### Initialize and create the Kubernetes Cluster

_Note: Run on master node_

``` 
sudo kubeadm init --ignore-preflight-errors all --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

#### [Install Calico](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises)

* Download the Calico networking manifest for the Kubernetes API datastore.
```
curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```
* Confirm 

``` watch kubectl get pods -n calico-system ```

* Remove the taints on the master so that you can schedule pods on it.

``` kubectl taint nodes --all node-role.kubernetes.io/master-```

```
kubectl get namespace
kubectl get pods --namespace=kube-system
kubectl describe -n kube-system pod calico-node-5894m
```

_[Error](https://stackoverflow.com/questions/54465963/calico-node-is-not-ready-bird-is-not-ready-bgp-not-established): calico/node is not ready: BIRD is not ready: BGP not established_
```
kubectl edit DaemonSet calico-node -n kube-system
vi calico.yaml
 # Specify interface
            - name: IP_AUTODETECTION_METHOD
              value: "interface=eth1"
              
kubectl set env daemonset/calico-node -n kube-system
IP_AUTODETECTION_METHOD=interface=eth1

```

####  Use the join commands on other VMs in the same network to join the cluster

``` 
kubeadm token create --print-join-command
\\ Example :
sudo kubeadm join 152.14.93.138:6443 --token xlq6yr.bw55wxm3rsbf2dt3 --ignore-preflight-errors all --discovery-token-ca-cert-hash sha256:b62f5079393b000052cb14652aae9a35888168c095c1ab466ced9584e6157b88
```
_Note: Run each time you start a new session to make worker node ready_

####  List the nodes part of the cluster using below command in master node:

``` sudo kubectl get nodes```

![image](https://user-images.githubusercontent.com/37688219/160314631-11a38fcf-f0b8-4ad7-9977-6a112c393fbe.png)

### Configuration steps for Openwhisk
---
####  Installation of the helm for setting up Openwhisk
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

#### Deployment of OpenWhisk
* We need to specify which of our nodes are core which operates the OpenWhisk control plane (the controller, kafka, zookeeeper, and couchdb pods) and invoker nodes which schedules and executes user containers. 
   * Label Cores nodes by running this command:
   
      ```kubectl label nodes <CORE_NODE_NAME> openwhisk-role=core```

   * Label Invoker nodes by running this command:
   
   ```kubectl label nodes <INVOKER_NODE_NAME> openwhisk-role=invoker```
   
   * Label all nodes as part of the cluster as Invoker nodes by running this command:

      ```kubectl label nodes --all openwhisk-role=invoker ```


* **Cluster configuration file setup (save as mycluster.yaml):**
```
whisk:
  ingress:
    type: NodePort
    apiHostName: 152.14.93.138
    apiHostPort: 31001
k8s:
  persistence:
    enabled: false
nginx:
  httpsNodePort: 31001
metrics:
  prometheusEnabled: true
metrics:
  userMetricsEnabled: true
```
* **Deploy OpenWhisk using the following commands:**
```
git clone https://github.com/apache/openwhisk-deploy-kube.git
sudo helm install owdev .openwhisk-deploy-kube/helm/openwhisk --namespace=openwhisk -f mycluster.yaml
```

* Check Logs:
``` 
kubectl get pods -n openwhisk
kubectl logs -n openwhisk owdev-init-couchdb-mqhbl
kubectl describe node master-node
kubectl get events --all-namespaces  --sort-by='.metadata.creationTimestamp'
```

* Opening shell for any particular pod
```
kubectl exec --stdin --namespace=openwhisk --tty owdev-apigateway-66486c84cd-lt4gj -- /bin/bash
```
* Uninstall Openwhisk
```
sudo helm uninstall owdev -n openwhisk
kubectl delete pod --grace-period=0 --force -n openwhisk owdev-init-couchdb-b75ms 
kubectl delete namespace other
```

_Error: Worker Node: (combined from similar events): failed to garbage collect required amount of images. Wanted to free 9628050227 bytes, but freed 1007351673 bytes_

```sudo docker run --rm --privileged -v /var/run/docker.sock:/var/run/docker.sock -v /etc:/etc:ro spotify/docker-gc ```

![image](https://user-images.githubusercontent.com/37688219/160316946-875c0a23-1a96-4a96-8977-677763249e5b.png)

### wsk CLI configuration
```
wget https://github.com/apache/openwhisk-cli/releases/download/latest/OpenWhisk_CLI-latest-linux-386.tgz
tar -xvf OpenWhisk_CLI-latest-linux-386.tgz
sudo mv wsk /usr/local/bin/wsk
```

* Check Installed

```wsk --help```
![image](https://user-images.githubusercontent.com/37688219/160317058-fe8464ad-3c7b-4b75-8b3c-d1d896c7b1c3.png)

```wsk property get```
![image](https://user-images.githubusercontent.com/37688219/160317147-879d8e3e-1c6d-4fc1-b7ed-7e7e93a160aa.png)
 
 * Check if it working or not by listing the installed packages:

```  wsk -i package list /whisk.system```
![image](https://user-images.githubusercontent.com/37688219/160317317-d56b3b1e-62f6-47fc-a7fb-3873cd8a928d.png)

### Sample function deployment
* Save the code to a file. For example, hello.js.

```
/**
 * Hello world as an OpenWhisk action.
 */
function main(params) {
    var name = params.name || 'World';
    return {payload:  'Hello, ' + name + '!'};
}
```

* create the action by entering this command:

``` wsk -i action create hello hello.js```
*  Invoke the action by entering the following commands.

``` wsk -i action invoke hello --result```

* Run this command for passing parameter:
``` wsk -i action invoke hello --result --param name Shakshi ```

![image](https://user-images.githubusercontent.com/37688219/160317706-8d773297-38b7-4b2d-b7f0-a6df4dbbe2f8.png)


