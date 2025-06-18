

#  Kubernetes Cluster Setup with Jenkins + Ansible on RHEL using kubeadm

This project demonstrates how to build a **Kubernetes cluster manually** using `kubeadm` on **Red Hat Enterprise Linux (RHEL)** and deploy **Jenkins** and **Ansible** as pods for a fully containerized CI/CD setup.

---

##  Prerequisites

- RHEL 9 (1 master + 1+ worker nodes)
- 2 vCPUs and 2 GB RAM minimum per node
- Docker + cri-dockerd
- Kubernetes tools: kubeadm, kubelet, kubectl
- Swap disabled
- Port `6443` open on master node (security group)

---

## File Structure

```
.
├── README.md
├── Dockerfile
├── ansible.cfg
├── inventory.ini
├── ansible-pod.yaml
├── jenkins-deployment.yaml
├── jenkins-pv.yaml
├── jenkins-pvc.yaml
├── Jenkinsfile
├── deploy-http.yml
```

## 🛠️ Setup Steps

### 🔧 Common Setup (All Nodes)

```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# Load kernel modules
echo 'br_netfilter' | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe br_netfilter

# System params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
````

---

## 🐳 Install Docker + cri-dockerd

```bash
sudo dnf install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
```

### cri-dockerd Setup

```bash
git clone https://github.com/Mirantis/cri-dockerd.git
cd cri-dockerd && mkdir bin
go build -o bin/cri-dockerd
sudo cp bin/cri-dockerd /usr/bin/
sudo cp -a packaging/systemd/* /etc/systemd/system/
sudo systemctl daemon-reexec
sudo systemctl enable --now cri-docker.service
```

---

## ☸️ Install Kubernetes Tools

```bash
# Disable SELinux
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Add Kubernetes repo
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

# Install tools
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

---

## 🧩 Initialize Master Node

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 📡 Install Network Add-on (Calico)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

---

## 🧩 Join Worker Nodes

On master node:

```bash
kubeadm token create --print-join-command
```

On worker node:

```bash
# Join command (append --v=5 for verbose)
sudo kubeadm join <MASTER_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_HASH> \
  --cri-socket=unix:///var/run/cri-dockerd.sock
```

---

## 🧪 Verify Cluster

```bash
kubectl get nodes
```

---

## 🧰 Jenkins + Ansible in Kubernetes

### Dockerfile (Custom Jenkins)

```Dockerfile
FROM jenkins/jenkins:lts
USER root
RUN apt update && apt install -y docker.io ansible sshpass && usermod -aG docker jenkins
USER jenkins
```

### Build & Push Image

```bash
docker build -t <dockerhub-username>/jenkins-ansible:v1 .
docker push <dockerhub-username>/jenkins-ansible:v1
```

---

## 📦 Deploy Jenkins & Ansible Pods

```bash
kubectl apply -f jenkins-pv.yaml
kubectl apply -f jenkins-pvc.yaml
kubectl apply -f jenkins-deployment.yaml
kubectl apply -f ansible-pod.yaml  # Optional
```

Access Jenkins: `http://<NodeIP>:30080`

---

## 🔐 Unlock Jenkins

```bash
kubectl exec -it <jenkins-pod> -- cat /var/jenkins_home/secrets/initialAdminPassword
```

---

## 🧪 Sample Jenkins Pipeline

```groovy
pipeline {
  agent any
  stages {
    stage('Clone Repo') {
      steps {
        git 'https://github.com/<your-repo>.git'
      }
    }
    stage('Run Ansible') {
      steps {
        sh 'ansible-playbook deploy-http.yml -i inventory.ini'
      }
    }
  }
}
```

---

## 🔐 SSH Setup for Ansible Control Node

Inside Jenkins/Ansible pod:

```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -q -N ""
ssh-copy-id -i ~/.ssh/id_rsa.pub root@<node-ip>
```

Repeat this for all nodes.

---
