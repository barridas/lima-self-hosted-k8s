# lima-self-hosted-k8s

Steps to set up a self-hosted K8s cluster on Mac with Lima:
Install Lima.
Install Lima using Homebrew:
Code

    brew install lima
Create Lima VMs.
Define and create one or more Lima VMs. You can create a single VM for a control plane and additional VMs for worker nodes. For example, to create an Ubuntu VM:
Code

    limactl start --name=k8s-master template://ubuntu
    limactl start --name=k8s-worker-1 template://ubuntu
Access Lima VMs.
Access the shell of your Lima VMs:
Code

    limactl shell k8s-master
Prepare VMs for Kubernetes.
Within each VM, perform the following: Disable swap.
Code

        sudo swapoff -a
        sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
Install container runtime (e.g., containerd):
Code

        sudo apt update
        sudo apt install -y containerd
        sudo systemctl enable --now containerd
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

    sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
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


To add Istio and Rook to your self-hosted Kubernetes cluster running on macOS with Lima, follow these steps after you've already set up the cluster with Calico:

🧩 Step 1: Install Istio
✅ Install istioctl on macOS
Shellbrew install istioctlShow more lines
✅ Deploy Istio inside the Lima VM
SSH into the Lima VM:
Shelllimactl shell kubernetesShow more lines
Then inside the VM:
Shellistioctl install --set profile=default -yShow more lines
✅ Enable automatic sidecar injection
Shellkubectl label namespace default istio-injection=enabledShow more lines
✅ Verify Istio components
Shellkubectl get pods -n istio-systemShow more lines
You should see components like istiod, istio-ingressgateway, etc.

🧩 Step 2: Install Rook (Ceph)
✅ Clone Rook repo inside the Lima VM
Shellgit clone --single-branch --branch master https://github.com/rook/rook.gitShow more lines
✅ Apply Rook manifests
Shellkubectl apply -f crds.yamlkubectl apply -f common.yamlkubectl apply -f csi-operator.yamlkubectl apply -f operator.yamlShow more lines
✅ Deploy a Ceph cluster
Shellkubectl apply -f cluster.yamlShow more lines
✅ Verify Rook components
Shellkubectl get pods -n rook-cephShow more lines
You should see pods like rook-ceph-operator, rook-ceph-mon, rook-ceph-osd, etc.

🧪 Optional: Create a StorageClass
To use Rook Ceph for persistent volumes:
Shellkubectl apply -f storageclass.yamlShow more lines
Then test with a PVC:
YAMLapiVersion: v1kind: PersistentVolumeClaimmetadata:  name: test-pvcspec:  accessModes:    - ReadWriteOnce  resources:    requests:      storage: 1Gi  storageClassName: rook-ceph-blockShow more lines

Would you like a script that automates all these steps or a .zip package with all the manifests and instructions?
