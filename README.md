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
``` sudo systemctl start docker ```

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
* Initialize and create the Kubernetes Cluster
