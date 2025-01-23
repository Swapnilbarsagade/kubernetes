Setting up a Kubernetes cluster using **kubeadm** on Ubuntu instances on AWS Cloud involves several steps. Here’s a detailed guide to help you:

---

### **1. Set Up AWS EC2 Instances**
1. **Launch EC2 Instances**:
   - Use at least two Ubuntu instances:
     - **Master node**: Handles the Kubernetes control plane.
     - **Worker node(s)**: Runs the application workloads.
   - Instance type: `t2.medium` or higher (for sufficient CPU and memory).
   - Ensure the instances are in the same **VPC** and **subnet**.

2. **Security Groups**:
   - Allow the following ports in your EC2 instances' security group:
     - `6443`: Kubernetes API server.
     - `2379-2380`: etcd server client API.
     - `10250`: Kubelet API.
     - `10251`: kube-scheduler.
     - `10252`: kube-controller-manager.
     - `30000-32767`: NodePort services.
   - Allow SSH access (`22`) from your local machine.

3. **Connect to Instances**:
   SSH into the instances:
   ```bash
   ssh -i <your-key.pem> ubuntu@<instance-ip>
   ```

---

### **2. Install Kubernetes and Docker**
Perform these steps on all nodes (master and worker):

1. **Update and Install Dependencies**:
   ```bash
   sudo apt-get update
   sudo apt-get install -y apt-transport-https ca-certificates curl
   ```

2. **Add Kubernetes GPG Key and Repository**:
   ```bash
   curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
   echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```


#### **Steps to Manually Install Kubernetes Components**

1. **Download Kubernetes Binaries**
   Run the following commands to download the `.deb` packages for the required components:
   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubeadm"
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubelet"
   ```

2. **Move the Binaries**
   Move the downloaded binaries to `/usr/local/bin` so they are available globally:
   ```bash
   sudo chmod +x kubectl kubeadm kubelet
   sudo mv kubectl kubeadm kubelet /usr/local/bin/
   ```

3. **Verify Installation**
   Confirm that the components are correctly installed:
   ```bash
   kubectl version --client
   kubeadm version
   kubelet --version
   ```

### **Install Required Dependencies Kubernetes requires some additional dependencies**
1. **Download the `cri-tools` Package**:
   Run the following commands to download the latest version of `cri-tools` from GitHub:
   ```bash
   curl -LO https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.28.0/crictl-v1.28.0-linux-amd64.tar.gz
   ```

2. **Extract and Install**:
   Extract the downloaded file and move the binary to `/usr/local/bin`:
   ```bash
   sudo tar -zxvf crictl-v1.28.0-linux-amd64.tar.gz -C /usr/local/bin
   ```

3. **Verify Installation**:
   Check if `crictl` is installed correctly:
   ```bash
   crictl --version
   ```

---

### **Install Other Dependencies**
The other required dependencies (`conntrack`, `socat`, `ebtables`, `ethtool`) should be available in the default repositories. Install them with:
```bash
sudo apt-get update
sudo apt-get install -y conntrack socat ebtables ethtool
```

4. **Install Docker and Enable and Start Docker**:
   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io
   sudo systemctl enable docker
   sudo systemctl start docker
   ```

5. **Disable Swap** (required by Kubernetes):
   ```bash
   sudo swapoff -a
   sudo sed -i '/swap/d' /etc/fstab
   ```

6. **Enable Required Kernel Modules**:
   ```bash
   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   br_netfilter
   EOF

   sudo modprobe br_netfilter

   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   EOF

   sudo sysctl --system
   ```

---

### **3. Initialize the Master Node**
Perform this step only on the master node:

1. **Run kubeadm init**:
   ```bash
   sudo kubeadm init --pod-network-cidr=192.168.0.0/16
   ```
   - `--pod-network-cidr` specifies the network range for the pod network.

2. **Set Up kubectl Access**:
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

3. **Copy the Join Command**:
   After initialization, `kubeadm` will provide a `kubeadm join` command. Copy this command, as you'll need it to add worker nodes.

---

### **4. Install a Pod Network Add-On**
On the master node, install a network add-on to enable pod communication. For example, using **Calico**:
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Verify the pods in the `kube-system` namespace are running:
```bash
kubectl get pods -n kube-system
```

---

### **5. Join Worker Nodes**
On each worker node, use the `kubeadm join` command provided earlier:
```bash
sudo kubeadm join <master-ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

Verify the worker nodes are joined:
```bash
kubectl get nodes
```

---

### **6. Deploy Your Application**
Now that your cluster is ready:
1. Write the **Deployment** and **Service** YAML files for your application (as described in the previous steps).
2. Apply the files:
   ```bash
   kubectl apply -f deployment.yaml
   kubectl apply -f service.yaml
   ```

---

### **7. Troubleshooting and Monitoring**
- Check the status of nodes and pods:
  ```bash
  kubectl get nodes
  kubectl get pods -o wide
  ```
- View logs for debugging:
  ```bash
  kubectl logs <pod-name>
  ```

---

### **8. Clean Up**
If you're done with the cluster, clean up resources:
- Reset the nodes:
  ```bash
  sudo kubeadm reset
  ```
- Delete the EC2 instances.

---

Let me know if you face any issues or need help with a specific step!