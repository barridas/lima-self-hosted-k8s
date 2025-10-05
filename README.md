# lima-self-hosted-k8s

Steps to set up a self-hosted K8s cluster on Mac with Lima:
Install Lima.
Install Lima using Homebrew:

    brew install lima
Create Lima VMs.
Define and create one or more Lima VMs. You can create a single VM for a control plane and additional VMs for worker nodes. For example, to create an almalinux-10 VM:

    limactl start --name=k8s-master template://almalinux-10
    limactl start --name=k8s-worker-1 template://almalinux-10
Access Lima VMs.
Access the shell of your Lima VMs:

    limactl shell k8s-master
Prepare VMs for Kubernetes.
Within each VM, perform the following: Disable swap.

        sudo swapoff -a
        sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
Install container runtime (e.g., containerd):

        sudo dnf update -y
        sudo dnf install -y containerd
        sudo systemctl enable --now containerd
Configure containerd with systemd cgroup driver (recommended for Kubernetes)

        sudo mkdir -p /etc/containerd
        containerd config default | sudo tee /etc/containerd/config.toml    
        sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

Configure kernel modules and sysctl parameters.
Add necessary modules and parameters for Kubernetes networking.
Code

        sudo modprobe overlay
        sudo modprobe br_netfilter
        echo "overlay" | sudo tee /etc/modules-load.d/containerd.conf
        echo "br_netfilter" | sudo tee -a /etc/modules-load.d/containerd.conf
        echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee /etc/sysctl.d/k8s.conf
        echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/k8s.conf
        sudo sysctl --system
Install kubeadm, kubelet, and kubectl:
On all VMs (master and workers):
Code

        sudo dnf update -y
        sudo dnf install -y curl gpg
        cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
        [kubernetes]
        name=Kubernetes
        baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
        enabled=1
        gpgcheck=1
        gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
        EOF
        
        sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

        sudo tee -a /etc/dnf/dnf.conf <<EOF
        exclude=kubelet kubeadm kubectl
        EOF
        sudo systemctl enable --now kubelet


Initialize the Kubernetes control plane (on the master VM only):
Code

    sudo kubeadm init --pod-network-cidr=10.244.0.0/16 # Use a suitable CIDR for your CNI
Follow the instructions provided by kubeadm init to configure kubectl for your user and to get the kubeadm join command for worker nodes.
Deploy a Container Network Interface (CNI) plugin (on the master VM):
Choose a CNI plugin (e.g., Flannel, Calico) and deploy it. For Flannel:
Code

    kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
    # For Calico kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
Join worker nodes to the cluster (on worker VMs):
Use the kubeadm join command obtained from the master node's initialization output. Verify the cluster.
On your Mac, outside the Lima VMs, ensure kubectl is configured to connect to your cluster. You might need to copy the kubeconfig file from the master VM to your ~/.kube/config directory on your Mac.
Code

    limactl cp k8s-master:~/.kube/config ~/.kube/config
    kubectl get nodes
This should show your master and worker nodes in a Ready state.
