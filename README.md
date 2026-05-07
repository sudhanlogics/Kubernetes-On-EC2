# Kubernetes Installation Manual
### Ubuntu 24.04 on AWS EC2 — From Zero to WordPress

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [System Preparation](#2-system-preparation)
3. [Install Docker & containerd](#3-install-docker--containerd)
4. [Install Kubernetes](#4-install-kubernetes)
5. [Initialize the Cluster](#5-initialize-the-cluster)
6. [Install Network Plugin (Calico)](#6-install-network-plugin-calico)
7. [Deploy WordPress Stack](#7-deploy-wordpress-stack)
8. [Access the Site](#8-access-the-site)
9. [Useful Commands](#9-useful-commands)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Prerequisites

- AWS EC2 instance — **Ubuntu 24.04 LTS**
- Instance type: **t3.medium** or higher (2 vCPU, 4GB RAM minimum)
- Storage: 20GB+
- The following ports open in your EC2 **Security Group**:

| Port  | Protocol | Purpose                  |
|-------|----------|--------------------------|
| 22    | TCP      | SSH access               |
| 80    | TCP      | HTTP                     |
| 6443  | TCP      | Kubernetes API server    |
| 30080 | TCP      | WordPress NodePort       |

---

## 2. System Preparation

SSH into your EC2 instance:

```bash
ssh -i your-key.pem ubuntu@<your-ec2-ip>
```

### 2.1 Update the System

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

### 2.2 Disable Swap

Kubernetes requires swap to be off:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 2.3 Enable IP Forwarding

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/k8s.conf

sudo sysctl --system
```

Verify:

```bash
cat /proc/sys/net/ipv4/ip_forward
# Expected output: 1
```

### 2.4 Load Required Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

---

## 3. Install Docker & containerd

### 3.1 Install containerd

```bash
sudo apt-get install -y containerd
```

### 3.2 Configure containerd

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup — required for Kubernetes
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

### 3.3 Start containerd

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd

# Verify
sudo systemctl status containerd
```

### 3.4 Install Docker (optional — for local image builds)

```bash
sudo apt-get install -y ca-certificates curl gnupg

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli docker-compose-plugin

sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
```

---

## 4. Install Kubernetes

### 4.1 Install Dependencies

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

### 4.2 Add Kubernetes Repository

> **Note:** The old `packages.cloud.google.com` repo is deprecated. Use the new official repo `pkgs.k8s.io`.

```bash
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
    sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
    sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Verify the repo was added:

```bash
cat /etc/apt/sources.list.d/kubernetes.list
# Expected:
# deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /
```

### 4.3 Install kubelet, kubeadm, kubectl

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Lock versions to prevent accidental upgrades
sudo apt-mark hold kubelet kubeadm kubectl
```

Verify:

```bash
kubectl version --client
kubeadm version
kubelet --version
```

---

## 5. Initialize the Cluster

### 5.1 Run kubeadm init

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

> This takes 2–5 minutes. Save the `kubeadm join` command printed at the end — you will need it to add worker nodes.

### 5.2 Set Up kubectl Access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

export KUBECONFIG=/etc/kubernetes/admin.conf
```

### 5.3 Allow Scheduling on Control Plane (Single Node)

By default, Kubernetes does not schedule pods on the control plane node. On a single node cluster, remove this restriction:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### 5.4 Verify Node

```bash
kubectl get nodes
# Expected (NotReady is normal before CNI is installed):
# NAME   STATUS     ROLES           AGE   VERSION
# k8s    NotReady   control-plane   1m    v1.30.x
```

---

## 6. Install Network Plugin (Calico)

The node stays `NotReady` until a CNI plugin is installed:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

Wait 2–3 minutes then verify:

```bash
# Watch pods come up
watch kubectl get pods -n kube-system

# Check node — should now show Ready
kubectl get nodes
# Expected:
# NAME   STATUS   ROLES           AGE   VERSION
# k8s    Ready    control-plane   5m    v1.30.x
```

---

## 7. Deploy WordPress Stack

### 7.1 Create Storage Directories on EC2

```bash
sudo mkdir -p /mnt/data/mysql
sudo mkdir -p /mnt/data/wordpress
```

### 7.2 Create the Manifest File

Create a file called `wordpress-manifest.yaml`:

```yaml
---
# NAMESPACE
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress

---
# SECRETS
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: wordpress
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: rootpassword
  MYSQL_DATABASE: wordpress
  MYSQL_USER: wp_user
  MYSQL_PASSWORD: wp_password

---
# CONFIGMAP — Nginx
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: wordpress
data:
  default.conf: |
    server {
        listen 80;
        server_name _;
        root /var/www/html;
        index index.php index.html;

        location / {
            try_files $uri $uri/ /index.php?$args;
        }

        location ~ \.php$ {
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param HTTP_PROXY "";
            fastcgi_read_timeout 300;
        }

        location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
            expires 30d;
            access_log off;
        }

        location ~ /\.(ht|git|env) { deny all; }
        location = /xmlrpc.php     { deny all; }
    }

---
# PERSISTENT VOLUME — MySQL
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data/mysql

---
# PERSISTENT VOLUME — WordPress
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data/wordpress

---
# PERSISTENT VOLUME CLAIM — MySQL
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi

---
# PERSISTENT VOLUME CLAIM — WordPress
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi

---
# DEPLOYMENT — MySQL
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_DATABASE
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
          readinessProbe:
            exec:
              command: ["mysqladmin", "ping", "-h", "localhost"]
            initialDelaySeconds: 20
            periodSeconds: 10
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc

---
# SERVICE — MySQL (Headless)
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
  clusterIP: None

---
# DEPLOYMENT — WordPress (PHP-FPM + Nginx)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      initContainers:
        - name: wait-for-mysql
          image: busybox
          command:
            - sh
            - -c
            - |
              until nc -z mysql 3306; do
                echo "Waiting for MySQL...";
                sleep 3;
              done
      containers:
        - name: php-fpm
          image: wordpress:6.5-php8.2-fpm
          env:
            - name: WORDPRESS_DB_HOST
              value: mysql:3306
            - name: WORDPRESS_DB_NAME
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_DATABASE
            - name: WORDPRESS_DB_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: wordpress-data
              mountPath: /var/www/html

        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: wordpress-data
              mountPath: /var/www/html
            - name: nginx-config
              mountPath: /etc/nginx/conf.d

      volumes:
        - name: wordpress-data
          persistentVolumeClaim:
            claimName: wordpress-pvc
        - name: nginx-config
          configMap:
            name: nginx-config

---
# SERVICE — WordPress (NodePort)
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  selector:
    app: wordpress
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

### 7.3 Apply the Manifest

```bash
kubectl apply -f wordpress-manifest.yaml
```

### 7.4 Verify Everything is Running

```bash
# Check PVs and PVCs are bound
kubectl get pv,pvc -n wordpress

# Check all pods are running
kubectl get pods -n wordpress

# Check services
kubectl get svc -n wordpress
```

All pods should show `Running` and PVCs should show `Bound`.

---

## 8. Access the Site

Open your browser and go to:

```
http://<your-ec2-public-ip>:30080
```

Complete the WordPress installation wizard.

---

## 9. Useful Commands

```bash
# Get all resources in wordpress namespace
kubectl get all -n wordpress

# Describe a pod (useful for debugging)
kubectl describe pod <pod-name> -n wordpress

# View pod logs
kubectl logs <pod-name> -n wordpress
kubectl logs <pod-name> -c nginx -n wordpress      # specific container
kubectl logs <pod-name> -c php-fpm -n wordpress

# Exec into a pod
kubectl exec -it <pod-name> -n wordpress -- bash

# Restart a deployment
kubectl rollout restart deployment/wordpress -n wordpress

# Delete and reapply manifest
kubectl delete -f wordpress-manifest.yaml
kubectl apply -f wordpress-manifest.yaml

# Check node status
kubectl get nodes
kubectl describe node <node-name>
```

---

## 10. Troubleshooting

### Pod stuck in Pending — Unbound PVC

```bash
# Create host directories
sudo mkdir -p /mnt/data/mysql
sudo mkdir -p /mnt/data/wordpress

# Check PVC status
kubectl get pvc -n wordpress
```

### Pod stuck in Pending — Control plane taint (single node)

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### API server connection refused

```bash
# Check kubelet
sudo systemctl status kubelet
sudo systemctl restart kubelet

# Check logs
journalctl -u kubelet -f
```

### containerd not running

```bash
sudo systemctl restart containerd
sudo systemctl status containerd
```

### Reset everything and start over

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/etcd $HOME/.kube
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
