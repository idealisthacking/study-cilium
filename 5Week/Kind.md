# kind 소개 : ‘도커 IN 도커 docker in docker’로 쿠버네티스 클러스터 환경을 구성 - [Link](https://kind.sigs.k8s.io/)
- kind or kubernetes in docker is a suite of tooling for local Kubernetes “clusters” where each “node” is a Docker container
- kind is targeted at testing Kubernetes , kind supports multi-node (including HA) clusters
- kind uses kubeadm to configure cluster nodes.
![[Pasted image 20250810213500.png]]

# kind 설치 : macOS 사용자

## Docker Desktop 설치 - [Link](https://docs.docker.com/desktop/install/mac-install/)
실습 환경 : Kind 사용하는 도커 엔진 리소스에 최소 vCPU 4, Memory 8GB 할당을 권고 - [링크](https://kind.sigs.k8s.io/docs/user/quick-start/#settings-for-docker-desktop)

## kind 및 툴 설치
 필수 툴 설치
```bash
# Install Kind
brew install kind
kind --version

# Install kubectl
brew install kubernetes-cli
kubectl version --client=true

## kubectl -> k 단축키 설정
echo "alias kubectl=kubecolor" >> ~/.zshrc

# Install Helm
brew install helm
helm version
```
(권장) 유용한 툴 설치
```bash
# 툴 설치
brew install krew
brew install kube-ps1
brew install kubectx

# kubectl 출력 시 하이라이트 처리
brew install kubecolor
echo "alias kubectl=kubecolor" >> ~/.zshrc
echo "compdef kubecolor=kubectl" >> ~/.zshrc
```


# kind 기본 사용 - 클러스터 배포 및 확인
(옵션) 별도 kubeconfig 지정 후 사용
회사에서 사용 중인 kubeconfig 가 있다면, 미리 kubeconfig 백업 후 kind 실습을 하시거나 혹은 아래 kubeconfig 변수 지정 사용 권장
```bash
# 방안1 : 환경변수 지정
export KUBECONFIG=/Users/<Username>/Downloads/kind/config

# 방안2 : 혹은 --kubeconfig ./config 지정 가능

# 클러스터 생성
kind create cluster

# kubeconfig 파일 확인
ls -l /Users/<Username>/Downloads/kind/config
-rw-------  1 <Username>  staff  5608  4 24 09:05 /Users/<Username>/Downloads/kind/config

# 파드 정보 확인
kubectl get pod -A

# 클러스터 삭제
kind delete cluster
unset KUBECONFIG
```

```bash
# 클러스터 배포 전 확인
docker ps

# Create a cluster with kind
kind create cluster

# 클러스터 배포 확인
kind get clusters
kind get nodes
kubectl cluster-info

# 노드 정보 확인
kubectl get node -o wide

# 파드 정보 확인
kubectl get pod -A
kubectl get componentstatuses

# 컨트롤플레인 (컨테이너) 노드 1대가 실행
docker ps
docker images

# kube config 파일 확인
cat ~/.kube/config
혹은
cat $KUBECONFIG # KUBECONFIG 변수 지정 사용 시

# nginx 파드 배포 및 확인 : 컨트롤플레인 노드인데 파드가 배포 될까요?
kubectl run nginx --image=nginx:alpine
kubectl get pod -owide

# 노드에 Taints 정보 확인
kubectl describe node | grep Taints
Taints:             <none>

# 클러스터 삭제
kind delete cluster

# kube config 삭제 확인
cat ~/.kube/config
혹은
cat $KUBECONFIG # KUBECONFIG 변수 지정 사용 시
```

# kind 로 k8s 배포 : macOS 사용자
## 기본 정보 확인
```bash
# 클러스터 배포 전 확인
docker ps

# Create a cluster with kind
kind create cluster --name myk8s --image kindest/node:v1.32.2 --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
  - containerPort: 30001
    hostPort: 30001
  - containerPort: 30002
    hostPort: 30002
  - containerPort: 30003
    hostPort: 30003
- role: worker
- role: worker
- role: worker
EOF


# 확인
kind get nodes --name myk8s
kubens default

# kind 는 별도 도커 네트워크 생성 후 사용 : 기본값 172.18.0.0/16
docker network ls
docker inspect kind | jq

# k8s api 주소 확인 : 어떻게 로컬에서 접속이 되는 걸까?
kubectl cluster-info

# 노드 정보 확인 : CRI 는 containerd 사용
kubectl get node -o wide

# 파드 정보 확인 : CNI 는 kindnet 사용
kubectl get pod -A -o wide

# 네임스페이스 확인 >> 도커 컨테이너에서 배운 네임스페이스와 다릅니다!
kubectl get namespaces

# 컨트롤플레인/워커 노드(컨테이너) 확인 : 도커 컨테이너 이름은 myk8s-control-plane , myk8s-worker/2/3 임을 확인
docker ps
docker images
docker exec -it myk8s-control-plane ss -tnlp

# 디버그용 내용 출력에 ~/.kube/config 권한 인증 로드
kubectl get pod -v6

# kube config 파일 확인
cat ~/.kube/config
혹은
cat $KUBECONFIG
```

## (참고) 클러스터 삭제
```bash
# 클러스터 삭제
kind delete cluster --name myk8s
docker ps
cat ~/.kube/config
```
