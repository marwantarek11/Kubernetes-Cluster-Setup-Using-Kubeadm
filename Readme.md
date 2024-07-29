# Kubernetes Cluster Setup Using Kubeadm on AWS
![1667359076133](https://github.com/user-attachments/assets/127806e8-0a8f-471b-b848-360b2f8246b9)

Step-by-step instructions to set up a Kubernetes cluster using `kubeadm` on AWS. The setup includes creating 1 master node and 2 worker nodes using Terraform to provision the EC2 instances. Additionally, we will cover the necessary prerequisites, including opening the required ports.

## Prerequisites

- AWS account with IAM permissions to create resources
- AWS CLI configured
- Terraform installed
- SSH key pair for accessing EC2 instances
- Basic knowledge of Kubernetes and AWS

## Required Ports

The following ports must be open for the Kubernetes cluster to function correctly:

![kuberetes-port-requirements-min](https://github.com/user-attachments/assets/e18b8e95-50e3-487f-8d2a-ba6162de40e2)

## Clone the Repository:
Clone this repository to your local machine:
```bash
git clone https://github.com/marwantarek11/Kubernetes-Cluster-Setup-Using-Kubeadm.git
```

## Terraform Setup


 **Initialize and apply the Terraform configuration:**

```sh
cd Terraform/instances  
terraform init
terraform apply
```
![image](https://github.com/user-attachments/assets/2b654189-a7f9-4de9-8f10-f98e3909aae6)


## Kubernetes Setup Using Kubeadm

1. **SSH into the master and worker nodes:**

    ```sh
    ssh -i test.pem ubuntu@<master-node-ip>
    ssh -i test.pem ubuntu@<worker-node1-ip>
    ssh -i test.pem ubuntu@<worker-node2-ip>
    ```

2. **Disable Swap & Add kernel Parameters**
   
   Execute beneath swapoff and sed command to disable swap. Make sure to run the following commands on all the nodes.

    ```sh
    sudo swapoff -a
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    ```
   Load the following kernel modules on all the nodes,
    
    ```bash
    sudo tee /etc/modules-load.d/containerd.conf <<EOF
    overlay
    br_netfilter
    EOF
    sudo modprobe overlay
    sudo modprobe br_netfilter
    ```
   Set the following Kernel parameters for Kubernetes, run beneath tee command
    ```bash
    sudo tee /etc/sysctl.d/kubernetes.conf <<EOT
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOT
    ```
   Reload the above changes, run
    ```bash
    sudo sysctl --system
    ```

3. **Install Containerd Runtime**

    In this guide, i used containerd runtime for our Kubernetes cluster. So, to install containerd, first install its dependencies.

    ```sh
    sudo apt install -y curl gnupg software-properties-common apt ca-certificates
    ```
    Enable docker repository
    ```bash
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    ```
    Now, run following apt command to install containerd.
    ```bash
    sudo apt update
    sudo apt install -y containerd.io
    ```
    Configure containerd so that it starts using systemd as cgroup.
    ```bash
    containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
    sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
    ```
    Restart and enable containerd service
    ```bash
    sudo systemctl restart containerd
    sudo systemctl enable containerd
    ```

4. **Add Apt Repository for Kubernetes:**
    
    Kubernetes package is not available in the default Ubuntu 22.04 package repositories. So we need to add Kubernetes repository. run following command to download public signing key,
    ```bash
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```
    Next, run following echo command to add Kubernetes apt repository.
    ```bash
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```

5. **Install Kubectl, Kubeadm and Kubelet:**

    Post adding the repositories, install Kubernetes components like kubectl, kubelet and Kubeadm utility on all the nodes. Execute following set of commands,
   
    ```bash
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

7. **Install Kubernetes Cluster on Ubuntu 22.04:**

    Now, we are all set to initialize Kubernetes cluster. Run the following Kubeadm command on the master node only.
     
    ```bash
    sudo kubeadm init --control-plane-endpoint=k8smaster.example.net
    ```
    After the initialization is complete, you will see a message with instructions on how to join worker nodes to the cluster. Make a note of the kubeadm join command for future reference.
    ![image](https://github.com/user-attachments/assets/16751ffb-4c03-4dff-bdc8-197d318ed811)
   
    So, to start interacting with cluster, run following commands on the master node,
    ```bash
     mkdir -p $HOME/.kube
     sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
     sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
    next, try to run following kubectl commands to view cluster and node status
    ```bash
    kubectl cluster-info
    kubectl get nodes
    ```
    ![image](https://github.com/user-attachments/assets/878f149b-0c20-4935-9436-ee884e3fac77)


7. **Join the worker nodes to the cluster:**

    On each worker node, use the kubeadm join command you noted down earlier after initializing the master node on step 6. It should look something like this:
   
    ```sh
    sudo kubeadm join 52.70.178.69:6443 --token eynjaj.kbhvstcg3qpjxw3z --discovery-token-ca-cert-hash sha256:2eef18a72824dcee9723c7d20708a61c04de6e72c577dd0615921717003ec10b 
    ```
    Output from both the worker nodes,
    ![image](https://github.com/user-attachments/assets/a26cda9c-9be9-4ae2-b600-18deb70b59a1)

    Above output from worker nodes confirms that both the nodes have joined the cluster.Check the nodes status from master node using kubectl command,

    ```bash
    kubectl get nodes
    ```
    ![image](https://github.com/user-attachments/assets/8eb1c692-bcbc-4e46-8963-ddb0ab113199)

    As we can see nodes status is ‘NotReady’, so to make it active. We must install CNI (Container Network Interface) or network add-on plugins like Calico,
    
8. **Install Calico Network Plugin:**

    A network plugin is required to enable communication between pods in the cluster. Run following kubectl command to install Calico network plugin from the master node,

    ```sh
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
    ```

    Verify the status of pods in kube-system namespace,
    ```bash
    kubectl get pods -n kube-system
    ```
    Output,
    ![image](https://github.com/user-attachments/assets/edb8f0b9-2828-46e3-a2f2-9c295215fd58)

    Great, above confirms that nodes are active node. Now, we can say that our Kubernetes cluster is functional.


## Verify the Cluster

Your Kubernetes cluster is now set up with 1 master node and 2 worker nodes on AWS. You can deploy applications and manage your cluster using `kubectl`.

let's Deploy Nginx App 

```bash
kubectl apply -f nginx-app.yml
```
![image](https://github.com/user-attachments/assets/233d6535-894c-4e7a-9a1d-f6d82cd52671)
![image](https://github.com/user-attachments/assets/ed58f4b3-9d66-486d-a6c5-8ff41ff6cff9)


## Troubleshooting

- If any issues arise during the setup, refer to the [Kubeadm Troubleshooting Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/).

## References

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Calico Documentation](https://docs.projectcalico.org/)
