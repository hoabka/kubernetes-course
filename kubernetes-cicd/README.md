![Overview](https://cdn.hashnode.com/res/hashnode/image/upload/v1610454337971/peuPjH7K0.png)  
  
## Task #1  
Trong task này yêu cầu setup một CI/CD pipeline để triển khai một dịch vụ sử dụng Jenkins CI, Kubernetes và ArgoCD với chiến lược GitOps
## Prequisites  
| Yêu cầu |  Version|  
|---|---|  
| System | 2+ CPU, 2GB Free Memory, 20GB Free Disk | 
| OS | Ubuntu 18.04 |  
| K8s cluster | latest | 
| [nginx-ingress controller](https://kubernetes.github.io/ingress-nginx/deploy/#minikube) | latest |
| [ArgoCD](https://argo-cd.readthedocs.io/en/stable/getting_started/) | latest | 
  
  
## 1. Setup Jenkins pipeline
### 1.1. Fork repo
- Khi triển khai CI/CD với chiến lược GitOps, chúng ta sẽ tách biệt repo cho việc lưu các cấu hình và repo souce code
- Fork 2 repo dưới đây về repo các nhân


| Repo |  URL|  Ý nghĩa|
|---|---|---|  
| `argo-cd`| [HTTP](https://gitlab.com/hoabka/argo-cd.git) hoặc [SSH](git@gitlab.com:hoabka/argo-cd.git)  | Chứa Helm chart của ứng dụng
| `sampleapp`| [HTTP](https://gitlab.com/hoabka/gitops-sampleapp.git) hoặc [SSH](git@gitlab.com:hoabka/gitops-sampleapp.git) | Chứa Source Code + Jenkinsfile  


### 1.2. Cài đặt Pipeline
#### 1.2.1. Cài đặt Credential
- Cài đặt các biến sau dưới dạng Credential 

| Tên biến |  Ý nghĩa|  
|---|---|  
| `REGISTRY_NAME`| Tên của Docker Registry để lưu Docker Image |
| `DOCKER_REGISTRY_PASSWORD`| Registry password | 
| `DOCKER_REGISTRY_USERNAME`| Registry Username | 
| `GITHUB_CREDS`| Giá trị lưu SSH private key để truy cập tới Gitlab | 
#### 1.2.2. Tạo Jenkinsfile
Tạo Jenkinsfile theo các bước sau.
**Step 0**:
- Kéo sampleapp repo về local để thực hiện chỉnh sửa
```bash
git clone YOUR_SAMPLEAPP_REPO
cd gitops-sampleapp
```
**Step 1**: 
- Tạo các biến môi trường để đọc các Credential tạo ở bước `1.2.1`


<details>
  <summary>Solution</summary>
  
  ```bash
environment {
 REGISTRY                 = credentials('REGISTRY_NAME')
 DOCKER_REGISTRY_PASSWORD = credentials('DOCKER_REGISTRY_PASSWORD')
 DOCKER_REGISTRY_USER     = credentials('DOCKER_REGISTRY_USERNAME')
 GITHUB_CREDS             = credentials('GITHUB_CREDS')
 }
  ```
</details>

**Step 2**: 
- Tạo một stage để lây `Commit ID`. Giá trị này sẽ dùng để đánh Tag cho Docker Image

<details>
  <summary>Solution</summary>
  
  ```javascript
stage('Get GIT_COMMIT') {
	steps {
		script {
			GIT_COMMIT = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
		}
	}
}
  ```
</details>

**Step 3**:  
- Tạo stage `Docker Build` để build Docker Image và Image được đánh tag theo format `BRANCH_NAME-GIT_COMMIT` sau đó đẩy Docker Image này tới Registry

<details>
  <summary>Solution</summary>
  
  ```bash
 stage('docker-build') {
	 options {
		 timeout(time: 10, unit: 'MINUTES')
	 }
	 steps {
		 sh '''#!/usr/bin/env bash
		 echo "Shell Process ID: $$"
		 docker login --user $DOCKER_REGISTRY_USER --	password $DOCKER_REGISTRY_PASSWORD
		 docker build --tag ${REGISTRY}/sampleapp:${BRANCH}-${GIT_COMMIT} .
		 docker push ${REGISTRY}/sampleapp:${BRANCH}-${GIT_COMMIT}
		 '''
	 }
 }
  ```
</details>

**Step 4**: 
- Tạo một stage để lấy clone `argo-cd` repo từ nhánh `main` để có cấu hình Helm Chart

<details>
  <summary>Solution</summary>
  
  ```bash
 stage('Deploy') {
	 steps {
		 git branch: 'main', credentialsId: ${GITHUB_CREDSs}, url: YOUR_SSH_REPO
	 }
 }
  ```
</details>

> Chú ý: Sửa biến **YOUR_SSH_REPO** thành REPO của bạn sau khi fork về

**Step 5**: 
- Tạo một stage để Deploy cho môi trường `Development`. Điều kiện trigger stage này là khi `BRANCH` đang ở là nhánh `dev`. 
- Ngoài ra cần update giá trị của `Registry` và `Docker Image` bạn đã build ở `Step 3` vào giá trị trong file `values-dev.yaml`của Helm Chart.
- Sau cùng commit các thay đổi này lên trên `argo-cd` repo.

<details>
  <summary>Solution</summary>
  
  ```bash
 stage('Deploy DEV') {
	 when {
		 branch 'dev'
	 }
	 script {
		 steps {
			 sh '''#!/usr/bin/env bash
			 echo "Shell Process ID: $$"
			 # Replace Repository and tag
			 cd ./argo-cd/sampleapp
			 sed -r "s/^(\s*repository\s*:\s*).*/\1${REGISTRY}\/sampleapp/" -i values-dev.yml
			 sed -r "s/^(\s*tag\s*:\s*).*/\1${BRANCH}-${GIT_COMMIT}/" -i values-dev.yml
			 git commit -am 'Publish new version' && git push || echo 'no changes'
			 '''
		 }
	 }
}
  ```
</details>

**Step 6**: 
- Làm tương tự `Step 5` nhưng với điều kiện là nhánh `prod` và thay đổi file `values-prod.yaml` 

<details>
  <summary>Solution</summary>
  
  ```bash
 stage('Deploy PROD') {
	 when {
		 branch 'prod'
	 }
	 script {
		 steps {
			 sh '''#!/usr/bin/env bash
			 echo "Shell Process ID: $$"
			 # Replace Repository and tag
			 cd ./argo-cd/sampleapp
			 sed -r "s/^(\s*repository\s*:\s*).*/\1${REGISTRY}\/sampleapp/" -i values-prod.yml
			 sed -r "s/^(\s*tag\s*:\s*).*/\1${BRANCH}-${GIT_COMMIT}/" -i values-prod.yml
			 git commit -am 'Publish new version' && git push || echo 'no changes'
			 '''
		 }
	 }
}
  ```
</details>

**Step 7**:
- Sau khi tạo xong `Jenkinsfile` thì commit lên `sampleapp` repo nhánh main

#### 1.2.3. Tạo Pipeline
- Xem hướng dẫn cấu hình multibranch pipeline [tại đây](https://github.com/hoabka/jenkins-course/blob/master/jenkins-multibranch/task1/README.md#2-t%E1%BA%A1o-jenkins-multibranch-pipeline)

## 2. Setup ArgoCD
### 2.1. Setup ArgoCD admin UI
```bash
# Cai dat argocd cli
$ wget https://github.com/argoproj/argo-cd/releases/download/v2.1.3/argocd-linux-amd64
$ sudo mv argocd-linux-amd64 /usr/local/bin/argocd && sudo chmod +x /usr/local/bin/argocd

# Su Expose argocd-server service su dung port-forwarding
$ sudo kubectl port-forward --address=0.0.0.0  svc/argocd-server -n argocd 8443:443 &
```

- Lấy user/password của `argocd admin`
```bash
$ kubectl -n argocd get secret argocd-initial-admin-secret -o  jsonpath="{.data.password}" | base64 -d
```
- Sau khi setup xong có thể sử dụng được argocd UI với địa chỉ `https://IP:8443` và user admin và password lấy được ở bước trên

![ArgoCD UIl](https://github.com/hoabka/kuberntes-course/kubernetes-cicd/images/argocdui.JPG)


### 2.2. Setup ArgoCD repo
- Setup ArgoCD repo để cho phép ArgoCD có thể có quyền truy cập tới thư mục chứa `Helm Chart` của ứng dụng. Trong bài lab là gitlab repo `argo-cd`

- ADD Repo

![Add repo](https://github.com/hoabka/kuberntes-course/kubernetes-cicd/images/addRepo.JPG)

- Connect tới Repo sự dụng SSH

![Connect SSH](https://github.com/hoabka/kuberntes-course/kubernetes-cicd/images/connectSSHRepo.JPG)

### 2.3. Setup ArgoCD application
- `Application` trong argocd là khái niệm tương đồng với ứng dụng bạn muốn triển khai.
- Tạo `Argo Application`

![Create App](https://github.com/hoabka/kuberntes-course/kubernetes-cicd/images/createapp_1.JPG)

- Add repo chứa `Helm Chart`

![Create App](https://github.com/hoabka/kuberntes-course/kubernetes-cicd/images/createapp_2.JPG)

- Chọn cụm K8s cluster, namespace để deploy application và chọn file `values` của `Helm chart`

![Create App](https://github.com/hoabka/kuberntes-course/kubernetes-cicd/images/createapp_3.JPG)

## 3. Deploy Application
### 3.1.  Update application version

```bash
# Tao dev branch tu main branch
$ git checkout -b dev

# Sua file code, tao version moi
$ sed -i 's/Nice to meet you/Nice to meet you v2/g' main.js

# Commit va push len dev branch
$ git push origin dev
```
### 3.2.  Chạy Jenkins pipeline
- Build Jenkins job nhánh `dev`

### 3.3.  Manual sync application trên argoCD
- Sử dụng giao diện của `argocd`
- Click `Sync` button

![Sync App](https://github.com/hoabka/kuberntes-course/kubernetes-cicd/images/syncApp.JPG)

- Sync status

![Sync Status](https://github.com/hoabka/kuberntes-course/kubernetes-cicd/images/syncAppStatus.JPG)

### 3.4.  Verify ứng dụng sau khi được deploy
```bash
# Verify service của ứng dụng
$ kubectl get svc
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
dev-sampleapp   NodePort    10.100.243.76   <none>        80:30695/TCP   2m45s

# Expose ứng dụng sử dụng port-forward
$ kubectl port-forward svc/dev-sampleapp 8080:80 &

# Kiem tra ung dung
$ curl localhost:8080
Hello GitOps Friends of mine! Nice to meet you V2
```
### 3.5.  Deploy PROD
- Tạo `argocd application` dành cho nhánh `prod`, tương tự như nhánh `dev` (`Phần  2.3. Setup ArgoCD application`)

```bash
# Checkout prod branch or tao moi tu nhanh main
$ git checkout -b prod
$ git merge dev
$ git push orgin prod
```
- Tiếp theo  chạy Jenkins job với nhánh là prod và Sync argocd tương tự dev

## 4. Troubleshooting