![Overview](https://github.com/hoabka/kubernetes-course/blob/master/kubernetes-overview/images/overview.jpg)  
## Task #1  
Trong task này, bạn được yêu cầu cài đặt Kubernetes Cluster sử dụng công cụ **minikube**
## Prequisites  
| Yêu cầu |  Version|  
|--|--|  
| System | 2+ CPU, 2GB Free Memory, 20GB Free Disk | 
| OS | Ubuntu 18.04 |  
| Docker| latest |  
  
  
## 1. Cài đặt công cụ và bootstrap cluster
### 1.1.  Cài đặt kubectl
- **kubectl** là công cụ cho phép người dùng sử dụng để tương tác với một K8s cluster (utility)
```bash
# Cập nhật thông tin các pakages và dependencies
$ sudo apt-get update

# Download kubectl binary
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Kiểm tra binary checksum
$ curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
$ echo "$(<kubectl.sha256) kubectl" | sha256sum --check

# Change mode file kubectl và add vào PATH
$ sudo chmod +x kubectl
$ sudo mv kubectl /usr/local/bin

# Kiểm tra xem cài đặt thành công hay chưa
$ kubectl version --client
```
### 1.2.  Cài đặt minikube
- **minikube** là công cụ cho phép khởi tạo một K8s cluster đơn giản và nhanh nhất. 
> **minikube** phù hợp cho việc tạo cluster để dùng với mục đích learning và development và KHÔNG nên dùng trong môi trường Production

```bash
# Download và Install minikube
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
$ sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
### 1.3.  Bootstrap cluster
```bash
# Bootstrap cluster sử dụng user có quyền Admin, và không dùng user root
$ minikube start

# Kiểm tra cluster xem đã được cài đặt thành công hay chưa
$ kubectl get po -A

# Kiểm tra thành phần của một cụm Cluster
$ kubectl get pod -n kube-system
```

### Completed Task #1
## 2. Troubleshooting