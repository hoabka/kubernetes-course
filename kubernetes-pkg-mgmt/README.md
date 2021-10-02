![Overview](https://helm.sh/img/helm.svg)  
  
## Task #1  
Trong task này, bạn được yêu cầu cài đặt **Helm** và thực hành với Helm
## Prequisites  
| Yêu cầu |  Version|  
|---|---|  
| System | 2+ CPU, 2GB Free Memory, 20GB Free Disk | 
| OS | Ubuntu 18.04 |  
| K8s cluster | latest | 
| [nginx-ingress controller for minikube](https://kubernetes.github.io/ingress-nginx/deploy/#minikube) | latest |
  
  
## 1. Cài đặt Helm
### 1.1. About Helm
- **Helm** là công cụ cho phép người dùng có thể đóng gói các resources, application chạy trên Kubernetes. 
- **Helm** giúp việc triển khai application, chia sẻ công cụ chạy trên K8s được dễ dàng hơn
### 1.2. Cài đặt Helm (< 3.0)
```bash
# Download Helm
$ wget https://get.helm.sh/helm-v2.17.0-linux-amd64.tar.gz

# Giai nen va set path
$ tar -xzvf helm-v2.17.0-linux-amd64.tar.gz
$ sudo mv linux-amd64/helm /usr/local/bin/helm

# Verify
$ helm version
```
### 1.3.  Khởi tạo Helm
```bash
# Tao Role-Based Access Control
$ kubectl create -f helm-rbac.yaml

# Init helm với Service Account
$ helm init --service-account tiller
```
## 2. Làm việc với Helm
### 2.1.  helm search
- `helm search` cho phép tìm kiếm một helm chart được chia sẻ
```bash
# Search redis chart
$ helm search redis

# Search mysql chart
$ helm search mysql

# Search nginx chart
$ helm search nginx
```
### 2.2.  helm list 
- `helm list` cho phép liêt kê các chart đã được cài đặt
```bash
$ helm list
```
### 2.3.  helm create
- `helm create` cho phép khởi tạo chart
```bash
$ helm create mychart
```
### 2.4.  helm install
- `helm install` cho phép khởi tạo chart
```bash
$ helm install mychart
```
## 3. Khởi tạo và cài đặt helm chart
### 3.1. Khởi tạo helm chart
```bash
# Create helm chart
$ helm create demo-chart
```
- Show tree của `demo-chart`
```
demo-chart/
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```
- Sửa `values.yaml` file để enable `ingress` và set `path` như dưới đây
```
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: demo-chart.local
      paths:
      - /
```
### 3.2. Install chart
```bash
$ helm install --name demo-chart ./demo-chart 
```
### 3.3. Verify and test
```bash
# Verify chart
helm ls | egrep demo-chart
# Verify POD and Service
$ kubect get pod
NAME                         READY   STATUS    RESTARTS   AGE
demo-chart-7cdf6f579-5wnr9   1/1     Running   0          30m

$ kubect get services
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
demo-chart      ClusterIP   10.107.250.180   <none>        80/TCP         30m

# Verify ingress
$ kubect get ingress
NAME         CLASS    HOSTS              ADDRESS        PORTS   AGE
demo-chart   <none>   demo-chart.local   192.168.49.2   80      28m
```
- Test dịch vụ đã được deploy thành công sử dụng **helm chart** hay chưa
```bash
$ curl `minikube ip` -H 'Host: demo-chart.local'
```
### Completed Task #1
## 2. Troubleshooting