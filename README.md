# bs23_Tasks
The goal is to install kubernetes cluster using vagrant and configuration management tool ansible. After that we will install Metric Server & k8s Dashboard.

## Installing Vagrant

Vagrant is an open-source software product for building and maintaining portable virtual software development environments; e.g., for VirtualBox, KVM, Hyper-V, Docker containers, VMware, and AWS. It tries to simplify the software configuration management of virtualization in order to increase development productivity.
To install Vagrant please follow the instructions bellow (I used ubuntu)

If VirtualBox is not installed on your system you can install it by running:
```
sudo apt update
```
```
sudo apt install virtualbox
```

The Vagrant package, which is available in Ubuntu’s repositories, is not regularly updated. We’ll download and install the latest version of Vagrant from the official Vagrant site.

At the time of writing this article, the latest stable version of Vagrant is version 2.2.9. Visit the Vagrant downloads page to see if there is a new version of Vagrant available.

Download the Vagrant package with wget :

```
curl -O https://releases.hashicorp.com/vagrant/2.2.9/vagrant_2.2.9_x86_64.deb
```
Once the file is downloaded, install it by typing:
```
sudo apt install ./vagrant_2.2.9_x86_64.deb
```

To verify that the installation was successful, run the following command that will print the Vagrant version:
```
vagrant --version
```

## Installing ansible
Ansible is an agentless automation tool that you install on a control node. From the control node, Ansible manages machines and other devices remotely (by default, over the SSH protocol)
Run the following command to install ansible on ubuntu
```
sudo apt install ansible
```
As a vagrant provider I have used Oracle virtual box.

## Vagrantfile
Check out the "Vagrantfile"

## Ansible playbook for Kubernetes master
Checkout the two files
```
master-node-ans-playbook.yml
worker-node-ans-playbook.yml
```

Run the command
```
vagrant up
```

Your 1 master 2 worker node kubernetes cluster will be installed. Now we will install kubernetes dashboard and metric server.

The version of the kubernetes installed: 1.22.2 Cgroup drive used = systemd

Dashboard Installaion ssh into your masternode by runnig this command vagrant ssh kube-master-node and then run the below snippet.
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
```

After that run "kubectl proxy" command from the node you want to view the dashboash You can see the Dashboard by this link: 
```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

## install Metric server

For this ssh into the master node using "vagrant ssh kube-master-node" and run this command. 
```
git clone https://github.com/kodekloudhub/kubernetes-metrics-server.git"
```
This will download all the files needed for metric server. After that go to the directory named Kubernetes-metrics-server and run 
```
kubectl apply -f .
```

This install all the necessary kubernetes objects for metric server. Now create some pods and wait for sometime to load the metric server. After a while run 
```
kubectl top pods
```
to view the resource consumption of your pod.
For Node resource consumption run
```
kubectl top nodes
```
