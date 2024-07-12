Pre-requisites  
1. Download and install virutual box https://www.virtualbox.org/wiki/Downloads
    sudo dpkg -i virtualbox-7.0_7.0.18-162988~Ubuntu~jammy_amd64.deb
    above migth result in errors during installation of virtualbox, To resolve the errors and install virtualbox run below command
    sudo apt -f install
2. Instsall vagrant to provision virtual machines
    wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
    echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
    sudo apt update && sudo apt install vagrant
3. Download the code repository using git clone (or download as zip)
4. Goto code reposistory directory and run command: vagrant up (this will provision 3 virtual machines named controlplane, node01 and node02)
5. Verify virtual machines by running command: vagrant status
6. Login to each virtual machine by running command: vagrant ss controlplane (or vagrant ssh node01 or vagrant ssh node02)


7. Cluster Installation:
        Installation using kubeadm (https://v1-29.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
    
        7.1 Installing container runtime containderd://v1-29.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/
        7.1.1 Enable IPV4 forwarding
            ----Run below on all nodes for Forwarding IPv4 ------------------
            cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
            overlay
            br_netfilter
            EOF
            
            sudo modprobe overlay
            sudo modprobe br_netfilter
            
            # sysctl params required by setup, params persist across reboots
            cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
            net.bridge.bridge-nf-call-iptables  = 1
            net.bridge.bridge-nf-call-ip6tables = 1
            net.ipv4.ip_forward                 = 1
            EOF
            
            # Apply sysctl params without reboot
            sudo sysctl --system
            -----------------------------
            

             7.1.2 Add Docker's official GPG key:
                sudo apt-get update
                sudo apt-get install ca-certificates curl
                sudo install -m 0755 -d /etc/apt/keyrings
                sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
                sudo chmod a+r /etc/apt/keyrings/docker.asc
            
            7.1.3 Add the repository to Apt sources:
                echo \
                  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
                  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
                  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
                sudo apt-get update
        
            7.1.4 Install the Docker packages:
                sudo apt-get install containerd.io
                systemctl status containerd (to verify containerd is active)
            7.1.5 Configuring the systemd cgroup driver
                sudo vi /etc/containerd/config.toml (dG to delete all existing lines, paste below lines,save)
                replace existing content of config.toml with below in config.toml file and save exit
                    version = 2
                    [plugins]
                      [plugins."io.containerd.grpc.v1.cri"]
                       [plugins."io.containerd.grpc.v1.cri".containerd]
                          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
                            [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
                              runtime_type = "io.containerd.runc.v2"
                              [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
                                SystemdCgroup = true
        
                7.1.5 Restart the containerd 
                    sudo systemctl restart containerd
  

            7.2 Installing kubeadm, kubelet and kubectl
                7.2.1 Update the apt package index and install packages needed to use the Kubernetes apt repository:
                    sudo apt-get update
                    sudo apt-get install -y apt-transport-https ca-certificates curl gpg
                7.2.2 Download the public signing key for the Kubernetes package repositories
                    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
                7.2.3 Add the appropriate Kubernetes apt repository
                    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
                7.2.4 Update the apt package index
                    sudo apt-get update
                    sudo apt-get install -y kubelet kubeadm kubectl
                    sudo apt-mark hold kubelet kubeadm kubectl
        
            7.3 Creating a cluster with kubeadm (https://v1-29.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
                Run below only on controlplane node only
                Get ip addr of controlplane ndoe: ip addr
                sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=replace_with_ip_addr_of_master
                run the commands (output of previous commands)to copy kube-config file to home directory
        
            7.4 Installing Addons(https://v1-29.docs.kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy)
                 Run below on master node only:
                 wget https://reweave.azurewebsites.net/k8s/v1.29/net.yaml
                 vi net.yaml
                 add below env variabels under container named weave
                 name: IPALLOC_RANGE
                 value: 10.244.0.0/16
                 kubectl create -f net.yaml
        
             7.5 Join the worker nodes in the cluster by runnig below on worker nodes
                kubeadm token create --print-join-command (to print the join command to be run on workder nodes)
                sudo replace_with_join_commands (only run on all worker nodes)
                verify by running a container: kubectl run nginx --image=nginx

8. Upgrading the cluster using Kubeadm
   
        8.1 Upgrade kubeadm version before upgrading kuberentes cluster (kubeadm follows same release versions as k8s)
   
           echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/
           sudo apt update
           sudo apt-cache madison kubeadm
           apt-get install kubeadm=1.30.0
           kubeadm upgrade plan v1.30.0 (it will give you a lot of good information, the current cluster version, the kubeadm tool version)
                   (It also tells you that after we upgrade the control plane components, you must manually upgrade the kubelet versions on each node)
           kubeadm upgrade apply v1.30.0
           apt-get install kubelet=1.30.0 (kubelete needs to be upgraded separately, on all nodes, as its not installed by kubeadm)
           systemctl daemon-reload
           systemctl restart kubelet
           kubectl uncordon controlplane
   