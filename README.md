# kubernetes-on-premise
Deploy production ready Kubernetes cluster using Kubespray and launch Metallb and Nginx-Ingress in this cluster. Kubernetes cluster over multiple datacenters will also be covered in this project. This project will show how to create a Kubernetes cluster, but I will cover what should be considered while create the preview/stage/production environments for the development purpose.

## Introduction and goal
Kubespray is a composition of Ansible playbooks, inventory, provisioning tools, and domain knowledge for generic Kubernetes cluster configuration management tasks. Kubespray provides:
* A highly available cluster
* Composable attributes(Choice of the network plugin for instance)
* Support for most popular Linux distributions

Kubespray can run on bare metal and most cloud, which is great if we need to have a hybrid cloud environment to deploy some applications on the public cloud.This project will walk through the steps it takes to launch a Kubernetes cluster from bate metal, including the issue that might come up during the setup.



## Prerequisites

### To be able to launch a cluster using Kubespray, there are some requirements to be satisfied:
* Network requirement for Kubernetes over datacenters: Add permission of GRE permission in the firewall and add route/vpn to connect datacenters.
* A block of free IP addresses that will be assigned to VMs and MetalLB.
- Hardware requirements:
  - Minimum requirement[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/]
  - For a large cluster[https://kubernetes.io/docs/setup/best-practices/cluster-large/#size-of-master-and-master-components]
* Minimum required version of Kubernetes is v1.14.
* Ansible v2.7.8 (or newer, but not 2.8.x) and python-netaddr is installed on the machine that will run Ansible commands.
* Jinja 2.9 (or newer) is required to run the Ansible Playbooks.
* The target servers must have access to the Internet in order to pull docker images.
- The target servers are configured to allow IPv4 forwarding.
  - With Vagrant VMs, this can be done through the Provisioning feature.
  - In bare metal, this can be automated by cloud-init with Terraform.
- The ssh key of the controlled server (where runs the ansible-playbook) must be copied to all the servers’ part of your inventory.
  - Done by cloud-init configuration with Terraform.
* For Kubernetes built on multiple data centers, enable the Generic Routing Encapsulation(GRE) on the VPN between data centers. Test it by running "mtr" commands between pods on different data centers.
* Check the firewall(iptables) rules on the host machine of Kubernetes nodes to make sure all VMs can communicate each other.
The firewalls are not managed on VMs, you'll need to implement your own rules the way you used to. in order to avoid any issue during deployment, you should disable your firewall.
* While creating Kubernetes cluster in the private cloud across multiple data centers, make sure VMs can communicate with each other on different data centers.
If Kubespray is running from a non-root user account, correct privilege escalation method should be configured in the target servers. Then the ansible_become flag or command parameters --become or -b should be specified.

## Launch Kubernetes via Kubespray

We will useanother machine as the Ansible contrl machine to launch the cluster among the nodes for the k8s cluster.
1. **Install [python/python3](https://linuxize.com/post/how-to-install-python-3-7-on-ubuntu-18-04/) and [pip/pip3](https://linuxize.com/post/how-to-install-pip-on-ubuntu-18.04/) in the Ansible control machine.**
2. **CLone the Kubespray git repository into the control machine.**
    ```
    sudo apt-get install git -y
    git clone https://github.com/kubernetes-sigs/kubespray.git
    ```
3. **Go to the Kubespray directory and install all dependency packages from "requirements.txt"(ansible, jinja2, netadd, pbr, hvac, jmespath, ruamel,yaml, etc.).**
   ```
   sudo apt-get install python3-pip
   sudo pip3  install -r contrib/inventory_builder/requirements.txt
   sudo pip install -r requirements.txt
   ```
4. **Copy "inventory/sample” as "inventory/mycluster"**
   ```
   cp -rfp inventory/sample inventory/mycluster
   ```
5. **Update the Ansible inventory file with inventory builder and change it if you need. Running the two commands below will generate a “inventory/mycluster/hosts.ini” file. You may choose to use hosts.yml file.**
   ```
   declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
    CONFIG_FILE=inventory/mycluster/hosts.ini python3 contrib/inventory_builder/inventory.py ${IPS[@]}
   ```
   An example hosts.ini file of 3 master + 3 work nodes is shown below:
    ```
    # ## Configure 'ip' variable to bind kubernetes services on a
    # ## different ip than the default iface
    # ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
    [all]
    node1 ansible_host=192.168.33.121  ip=192.168.33.121 etcd_member_name=etcd1
    node2 ansible_host=192.168.33.122  ip=192.168.33.122 etcd_member_name=etcd2
    node3 ansible_host=192.168.33.123  ip=192.168.33.123 etcd_member_name=etcd3
    node4 ansible_host=192.168.33.124  ip=192.168.33.124
    node5 ansible_host=192.168.33.125  ip=192.168.33.125 
    node6 ansible_host=192.168.33.126  ip=192.168.33.126

    ## fix ubuntu18.04 problem
    #[all:vars]
    #ansible_user=ubuntu
    #ansible_python_interpreter=/usr/bin/python3
    #kubelet_cgroup_driver=cgroupfs 

    # ## configure a bastion host if your nodes are not directly reachable
    # bastion ansible_host=x.x.x.x ansible_user=some_user

    [kube-master]
    node1
    node2
    node3

    [etcd]
    node1
    node2
    node3

    [kube-node]
    node4
    node5
    node6

    [calico-rr]

    [k8s-cluster:children]
    kube-master
    kube-node
    calico-rr
    ```
6. **Review and modify parameters under "inventory/mycluster/group_vars" according to your environment and requirements**
   ```
   vim inventory/mycluster/group_vars/all/all.yml
   vim inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml
   ```
   **In “inventory/mycluster/group_vars/all/all.yml”**, uncomment the following statement to enable metrics to fetch the cluster resource utilization data. With this, HPAs will not work. Check more configurable parameters in Kubespray. K8s components require a loadbalancer to access the apiservers via a reverse proxy. Kubespray includes support for an nginx-based proxy that resides on each non-master Kubernetes node. This is referred to as localhost loadbalancing. It is less efficient than a dedicated load balancer because it creates extra health checks on the Kubernetes apiserver, but is more practical for scenarios where an external LB or virtual IP management is inconvenient. This option is configured by the variable loadbalancer_apiserver_localhost (defaults to True. Or False, if there is an external loadbalancer_apiserver defined).
   ```
   ## The read-only port for the Kubelet to serve on with no authentication/authorization. Uncomment to enable.
   kube_read_only_port: 10255

   ## Internal loadbalancers for apiservers
   loadbalancer_apiserver_localhost: true

   ## valid options are "nginx" or "haproxy" 
   loadbalancer_apiserver_type: nginx # valid values "nginx" or "haproxy"
   ```
   **In "inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml".**
   ```
   # configure arp_ignore and arp_announce to avoid answering ARP queries from kube-ipvs0 interface
   # must be set to true for MetalLB to work
   kube_proxy_strict_arp: false
   ```
7. **Configure the ansible user and private_key_file in ansible.cfg file.**
8. **If needed, change timeout for different ansible roles, for example, Kubeadm, etcd, API server. You can find those files inside /kubespray/roles directory.**
9. **Deploy Kubespray with Ansible Playbook**
   Run the playbook as root. The option `--become` is required, as for example writing SSL keys in /etc/, installing packages and interacting with various systemd daemons. Without --become the playbook will fail to run!
   ```
   ansible-playbook -i inventory/mycluster/hosts.ini --become --become-user=root cluster.yml
   ```
   This command will take around 30 minutes to finish and launch the k8s cluster on 6 VMs.
   
   **Tips:Speed up downloading binaries and containers.**  
   By default, each node downloads binaries and container images on its own, which can sometimes cause the failure of ansible-playbook if the network of the node is slow and trigger timeout.
   There is also a "pull once, push many" mode as well:
     - Setting download_run_once: True will make kubespray download container images and binaries only once and then push them to the cluster nodes. The default download delegate node is the first kube-master.
     - Set download_localhost: True to make localhost the download delegate. This can be useful if cluster nodes cannot access external addresses. To use this requires that docker is installed and running on the ansible master and that the current user is either in the docker group or can do passwordless sudo, to be able to access docker.
10. **Access k8s cluster viakubectl from the console by copy the /etc/kubernetes/admin.conf file to $HOME/.kube/config in the master node.**  
    To access the k8s cluster from your local machine, copy the contents of the /etc/kubernetes/admin.conf file into $HOME/.kube/config file in your machine.
    ```
    mkdir -p $HOME/.kube; sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
    Verify kubectl is working by:  
    ```
    kubectl get nodes
    kubectl cluster-info
    ```
    By default, kubectl looks for a file named config in the $HOME/.kube directory. You can specify other kubeconfig files by setting the KUBECONFIG environment variable or by setting the --kubeconfig=/path/to/your-config flag.
11. **Kubernetes cluster management: Adding/replacing a node, Cluster upgrades**  
    - [Adding/replacing a node](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/nodes.md)
    - [Upgrading Kubernetes cluster version](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/upgrades.md)

## Install MetalLB to provide load-balancer implementation

[MetalLB](https://metallb.universe.tf/concepts/) hooks into your Kubernetes cluster, and provides a network load-balancer implementation. In short, it allows you to create Kubernetes services of type “LoadBalancer” in clusters that don’t run on a cloud provider, and thus cannot simply hook into paid products to provide load-balancers.  

In the internal private network, we allocate internal IP addresses to LoadBalancer then associate the internal IP addresses with available public IPv4 addresses by port forwarding in the firewall to expose it to public.

An example manifest to deploy and configure MetalLB is located in kubernetes-on-premise/MetalLB directory. I use [kustomize](https://kustomize.io/), a Kubernetes native templating management tool, to manage Kubernetes deployment files. Take a look at kustomize and understand its base and overlay concepts before moving to the next part.

We will use MetalLB in layer 2 mode. In layer 2 mode, one node assumes the responsibility of advertising a service to the local network. From the network’s perspective, it simply looks like that machine has multiple IP addresses assigned to its network interface.

Under the hood, MetalLB responds to ARP requests for IPv4 services, and NDP requests for IPv6.

The major advantage of the layer 2 mode is its universality: it will work on any ethernet network, with no special hardware required, not even fancy routers.

First, modify the layer2-config.yaml file to specify the address-pools for MetalLB to allocate. Later, we can request an IP address from one address-pool by defining Kubernetes "LoadBalancer" type service.

Then, run `kubectl apply -k /path/to/metallb/overlays/your-layer/` to install the MetalLB.
