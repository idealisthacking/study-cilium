>[!note] 현재 1.18 버전의 ClusterMesh 분산 동작이 조금 이상한 것으로 보입니다. 스터디에서는 1.17.6 버전을 통해 진행합니다.

## kind k8s 클러스터 west, east 배포 - [Docs](https://docs.cilium.io/en/latest/installation/kind/)
```bash
#
kind create cluster --name west --image kindest/node:v1.33.2 --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000 # sample apps
    hostPort: 30000
  - containerPort: 30001 # hubble ui
    hostPort: 30001
- role: worker
  extraPortMappings:
  - containerPort: 30002 # sample apps
    hostPort: 30002
networking:
  podSubnet: "10.0.0.0/16"
  serviceSubnet: "10.2.0.0/16"
  disableDefaultCNI: true
  kubeProxyMode: none
EOF


# 설치 확인
kubectl ctx
kubectl get node 
kubectl get pod -A

# 노드에 기본 툴 설치
docker exec -it west-control-plane sh -c 'apt update && apt install tree psmisc lsof wget net-tools dnsutils tcpdump ngrep iputils-ping git -y'
docker exec -it west-worker sh -c 'apt update && apt install tree psmisc lsof wget net-tools dnsutils tcpdump ngrep iputils-ping git -y'

#
kind create cluster --name east --image kindest/node:v1.33.2 --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31000 # sample apps
    hostPort: 31000
  - containerPort: 31001 # hubble ui
    hostPort: 31001
- role: worker
  extraPortMappings:
  - containerPort: 31002 # sample apps
    hostPort: 31002
networking:
  podSubnet: "10.1.0.0/16"
  serviceSubnet: "10.3.0.0/16"
  disableDefaultCNI: true
  kubeProxyMode: none
EOF

# 설치 확인
kubectl config get-contexts 
CURRENT   NAME        CLUSTER     AUTHINFO    NAMESPACE
*         kind-east   kind-east   kind-east   
          kind-west   kind-west   kind-west 

kubectl config set-context kind-east
kubectl get node -v=6 --context kind-east
kubectl get node -v=6
kubectl get node -v=6 --context kind-west
cat ~/.kube/config

kubectl get pod -A
kubectl get pod -A --context kind-west

# 노드에 기본 툴 설치
docker exec -it east-control-plane sh -c 'apt update && apt install tree psmisc lsof wget net-tools dnsutils tcpdump ngrep iputils-ping git -y'
docker exec -it east-worker sh -c 'apt update && apt install tree psmisc lsof wget net-tools dnsutils tcpdump ngrep iputils-ping git -y'

```

## alias 설정
```bash
# alias 설정
alias kwest='kubectl --context kind-west'
alias keast='kubectl --context kind-east'

# 확인
kwest get node -owide
keast get node -owide

```

## (옵션) kind docker network 에 테스트용 PC(실제로는 컨테이너) 배포 : k8s join 되지 않은 클라이언트 단말 역할, k8s 외부에서 내부 진입 테스트용
```bash
# mypc 컨테이너 기동 : 윈도우 경우 IP 지정 에러 발생 시, IP 지정 옵션 빼고 기동
## 혹시 docker 대신 다른 툴 orbstack 등 사용한다면, docker network 가 172.18.0.0/16 아닐 수 있으니, 해당 툴의 대역 확인해서 사용.
docker run -d --rm --name mypc --network kind --ip 172.18.0.100 nicolaka/netshoot sleep infinity # macOS
docker run -d --rm --name mypc --network kind nicolaka/netshoot sleep infinity # Windows
docker ps


# kind network 중 컨테이너(노드) IP(대역) 확인
docker ps -q | xargs docker inspect --format '{{.Name}} {{.NetworkSettings.Networks.kind.IPAddress}}'
/west-control-plane 172.18.0.2
/east-control-plane 172.18.0.3
/mypc 172.18.0.100

# 동일한 docker network(kind) 내부에서 컨테이너 이름 기반 도메인 통신 가능 확인!
docker exec -it mypc ping -c 1 172.18.0.2
docker exec -it mypc ping -c 1 172.18.0.3
docker exec -it mypc ping -c 1 west-control-plane
docker exec -it mypc ping -c 1 east-control-plane

#
docker exec -it west-control-plane ping -c 1 east-control-plane
docker exec -it east-control-plane ping -c 1 west-control-plane

docker exec -it west-control-plane ping -c 1 mypc
docker exec -it east-control-plane ping -c 1 mypc
```

# 도전과제6 kind 와 같은 네트워크로 frr 컨테이너를 배포(bgp config 주입)하여, cilium 과 BGP(ECMP) + LB IPAM 로 인입 설정해보기

^7f1174


# Cilium CNI 배포 : ClusterMesh- [Docs](https://docs.cilium.io/en/stable/network/clustermesh/clustermesh/) , [Blog](https://nomadxd.github.io/blog/multi-cluster-networking-with-cilium-cluster-mesh)
```bash
# cilium cli 설치
brew install cilium-cli # macOS
https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli # Windwos WSL2 , 아래 명령어로 설치
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}


# cilium cli 로 cilium cni 설치해보기 : dry-run
cilium install --version 1.17.6 --set ipam.mode=kubernetes \
--set kubeProxyReplacement=true --set bpf.masquerade=true \
--set endpointHealthChecking.enabled=false --set healthChecking=false \
--set operator.replicas=1 --set debug.enabled=true \
--set routingMode=native --set autoDirectNodeRoutes=true --set ipv4NativeRoutingCIDR=10.0.0.0/16 \
--set ipMasqAgent.enabled=true --set ipMasqAgent.config.nonMasqueradeCIDRs='{10.1.0.0/16}' \
--set cluster.name=west --set cluster.id=1 \
--context kind-west --dry-run-helm-values

# cilium cli 로 cilium cni 설치
cilium install --version 1.17.6 --set ipam.mode=kubernetes \
--set kubeProxyReplacement=true --set bpf.masquerade=true \
--set endpointHealthChecking.enabled=false --set healthChecking=false \
--set operator.replicas=1 --set debug.enabled=true \
--set routingMode=native --set autoDirectNodeRoutes=true --set ipv4NativeRoutingCIDR=10.0.0.0/16 \
--set ipMasqAgent.enabled=true --set ipMasqAgent.config.nonMasqueradeCIDRs='{10.1.0.0/16}' \
--set cluster.name=west --set cluster.id=1 \
--context kind-west

watch kubectl get pod -n kube-system --context kind-west


# dry-run
cilium install --version 1.17.6 --set ipam.mode=kubernetes \
--set kubeProxyReplacement=true --set bpf.masquerade=true \
--set endpointHealthChecking.enabled=false --set healthChecking=false \
--set operator.replicas=1 --set debug.enabled=true \
--set routingMode=native --set autoDirectNodeRoutes=true --set ipv4NativeRoutingCIDR=10.1.0.0/16 \
--set ipMasqAgent.enabled=true --set ipMasqAgent.config.nonMasqueradeCIDRs='{10.0.0.0/16}' \
--set cluster.name=east --set cluster.id=2 \
--context kind-east --dry-run-helm-values

cilium install --version 1.17.6 --set ipam.mode=kubernetes \
--set kubeProxyReplacement=true --set bpf.masquerade=true \
--set endpointHealthChecking.enabled=false --set healthChecking=false \
--set operator.replicas=1 --set debug.enabled=true \
--set routingMode=native --set autoDirectNodeRoutes=true --set ipv4NativeRoutingCIDR=10.1.0.0/16 \
--set ipMasqAgent.enabled=true --set ipMasqAgent.config.nonMasqueradeCIDRs='{10.0.0.0/16}' \
--set cluster.name=east --set cluster.id=2 \
--context kind-east

watch kubectl get pod -n kube-system --context kind-east


# 확인
kwest get pod -A && keast get pod -A
cilium status --context kind-west
cilium status --context kind-east
cilium config view --context kind-west
cilium config view --context kind-east
kwest exec -it -n kube-system ds/cilium -- cilium status --verbose
keast exec -it -n kube-system ds/cilium -- cilium status --verbose

kwest -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg bpf ipmasq list
keast -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg bpf ipmasq list

# coredns 확인 : 둘 다, cluster.local 기본 도메인 네임 사용 중
kubectl describe cm -n kube-system coredns --context kind-west | grep kubernetes
    kubernetes cluster.local in-addr.arpa ip6.arpa {

kubectl describe cm -n kube-system coredns --context kind-west | grep kubernetes
    kubernetes cluster.local in-addr.arpa ip6.arpa {

# k9s 사용 시
k9s --context kind-west
k9s --context kind-east


# 삭제 시
# cilium uninstall --context kind-west
# cilium uninstall --context kind-east
```

# Setting up Cluster Mesh - [Docs](https://docs.cilium.io/en/stable/network/clustermesh/clustermesh/)
```bash
# 라우팅 정보 확인
docker exec -it west-control-plane ip -c route
docker exec -it west-worker ip -c route
docker exec -it east-control-plane ip -c route
docker exec -it east-worker ip -c route


# Specify the Cluster Name and ID : 이미 설정 되어 있음

# Shared Certificate Authority
keast get secret -n kube-system cilium-ca
keast delete secret -n kube-system cilium-ca

kubectl --context kind-west get secret -n kube-system cilium-ca -o yaml | \
kubectl --context kind-east create -f -

keast get secret -n kube-system cilium-ca


# 모니터링 : 신규 터미널 2개
cilium clustermesh status --context kind-west --wait  
cilium clustermesh status --context kind-east --wait


# Enable Cluster Mesh : 간단한 실습 환경으로 NodePort 로 진행
cilium clustermesh enable --service-type NodePort --enable-kvstoremesh=false --context kind-west
cilium clustermesh enable --service-type NodePort --enable-kvstoremesh=false --context kind-east
 

# 32379 NodePort 정보 : clustermesh-apiserver 서비스 정보
kwest get svc,ep -n kube-system clustermesh-apiserver --context kind-west
NAME                            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/clustermesh-apiserver   NodePort   10.2.216.182   <none>        2379:32379/TCP   65s

NAME                              ENDPOINTS         AGE
endpoints/clustermesh-apiserver   10.0.0.195:2379   65s # 대상 파드는 clustermesh-apiserver 파드 IP

kwest get pod -n kube-system -owide | grep clustermesh


keast get svc,ep -n kube-system clustermesh-apiserver --context kind-east
NAME                            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/clustermesh-apiserver   NodePort   10.3.252.188   <none>        2379:32379/TCP   43s

NAME                              ENDPOINTS         AGE
endpoints/clustermesh-apiserver   10.1.0.206:2379   65s # 대상 파드는 clustermesh-apiserver 파드 IP

keast get pod -n kube-system -owide | grep clustermesh


# 모니터링 : 신규 터미널 2개
watch -d "cilium clustermesh status --context kind-west --wait"
watch -d "cilium clustermesh status --context kind-east --wait"


# Connect Clusters
cilium clustermesh connect --context kind-west --destination-context kind-east

# 확인
cilium clustermesh status --context kind-west --wait
cilium clustermesh status --context kind-east --wait

# 
kubectl exec -it -n kube-system ds/cilium -c cilium-agent --context kind-west -- cilium-dbg troubleshoot clustermesh
kubectl exec -it -n kube-system ds/cilium -c cilium-agent --context kind-east -- cilium-dbg troubleshoot clustermesh

# 확인
kwest get pod -A && keast get pod -A
cilium status --context kind-west
cilium status --context kind-east
cilium clustermesh status --context kind-west
cilium clustermesh status --context kind-east
cilium config view --context kind-west
cilium config view --context kind-east
kwest exec -it -n kube-system ds/cilium -- cilium status --verbose
keast exec -it -n kube-system ds/cilium -- cilium status --verbose
ClusterMesh:   1/1 remote clusters ready, 0 global-services
   east: ready, 2 nodes, 4 endpoints, 3 identities, 0 services, 0 MCS-API service exports, 0 reconnections (last: never)
   └  etcd: 1/1 connected, leases=0, lock leases=0, has-quorum=true: endpoint status checks are disabled, ID: c6ba18866da7dfd8
   └  remote configuration: expected=true, retrieved=true, cluster-id=2, kvstoremesh=false, sync-canaries=true, service-exports=disabled
   └  synchronization status: nodes=true, endpoints=true, identities=true, services=true

#
helm get values -n kube-system cilium --kube-context kind-west 
...
cluster:
  id: 1
  name: west
clustermesh:
  apiserver:
    kvstoremesh:
      enabled: false
    service:
      type: NodePort
    tls:
      auto:
        enabled: true
        method: cronJob
        schedule: 0 0 1 */4 *
  config:
    clusters:
    - ips:
      - 172.18.0.4
      name: east
      port: 32379
    enabled: true
  useAPIServer: true
...

helm get values -n kube-system cilium --kube-context kind-east 
...
cluster:
  id: 2
  name: east
clustermesh:
  apiserver:
    kvstoremesh:
      enabled: false
    service:
      type: NodePort
    tls:
      auto:
        enabled: true
        method: cronJob
        schedule: 0 0 1 */4 *
  config:
    clusters:
    - ips:
      - 172.18.0.3
      name: west
      port: 32379
    enabled: true
  useAPIServer: true
...


# 라우팅 정보 확인 : 클러스터간 PodCIDR 라우팅 주입 확인!
docker exec -it west-control-plane ip -c route
docker exec -it west-worker ip -c route
docker exec -it east-control-plane ip -c route
docker exec -it east-worker ip -c route

```

# Hubble enable
```bash
# 설정
helm upgrade cilium cilium/cilium --version 1.17.6 --namespace kube-system --reuse-values \
--set hubble.enabled=true --set hubble.relay.enabled=true --set hubble.ui.enabled=true \
--set hubble.ui.service.type=NodePort --set hubble.ui.service.nodePort=30001 --kube-context kind-west
kwest -n kube-system rollout restart ds/cilium

## 혹은 cilium hubble enable --ui --relay --context kind-west
## kubectl --context kind-west patch svc -n kube-system hubble-ui -p '{"spec": {"type": "NodePort", "ports": [{"port": 80, "targetPort": 8081, "nodePort": 30001}]}}'


# 설정
helm upgrade cilium cilium/cilium --version 1.17.6 --namespace kube-system --reuse-values \
--set hubble.enabled=true --set hubble.relay.enabled=true --set hubble.ui.enabled=true \
--set hubble.ui.service.type=NodePort --set hubble.ui.service.nodePort=31001 --kube-context kind-east
kwest -n kube-system rollout restart ds/cilium

## 혹은 cilium hubble enable --ui --relay --context kind-east
## kubectl --context kind-east patch svc -n kube-system hubble-ui -p '{"spec": {"type": "NodePort", "ports": [{"port": 80, "targetPort": 8081, "nodePort": 31001}]}}'


# 확인
kwest get svc,ep -n kube-system hubble-ui --context kind-west
keast get svc,ep -n kube-system hubble-ui --context kind-east

# hubble-ui 접속 시
open http://localhost:30001
open http://localhost:31001

```


# west 파드 ↔ east 파드간 직접 통신 확인 with static routing : 파드간 직접 통신 tcpdump 로 확인
```bash
#
cat << EOF | kubectl apply --context kind-west -f -
apiVersion: v1
kind: Pod
metadata:
  name: curl-pod
  labels:
    app: curl
spec:
  containers:
  - name: curl
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
EOF

cat << EOF | kubectl apply --context kind-east -f -
apiVersion: v1
kind: Pod
metadata:
  name: curl-pod
  labels:
    app: curl
spec:
  containers:
  - name: curl
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
EOF
```
```bash

# 확인
kwest get pod -A && keast get pod -A
kwest get pod -owide && keast get pod -owide
NAME       READY   STATUS    RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
curl-pod   1/1     Running   0          43s   10.0.0.144   west-control-plane   <none>           <none>
NAME       READY   STATUS    RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
curl-pod   1/1     Running   0          36s   10.1.0.128   east-control-plane   <none>           <none>

#
kubectl exec -it curl-pod --context kind-west -- ping -c 1 10.1.0.128
64 bytes from 10.1.0.128: icmp_seq=1 ttl=62 time=0.877 ms

kubectl exec -it curl-pod --context kind-west -- ping 10.1.0.128

# 목적지 파드에서 tcpdump 로 확인 : NAT 없이 직접 라우팅.
kubectl exec -it curl-pod --context kind-east -- tcpdump -i eth0 -nn

# 목적지 k8s 노드?에서 icmp tcpdump 로 확인 : 다른곳 경유하지 않고 직접 노드에서 파드로 인입 확인
docker exec -it east-control-plane tcpdump -i any icmp -nn
docker exec -it east-worker tcpdump -i any icmp -nn

#
kubectl exec -it curl-pod --context kind-east -- ping -c 1 10.0.0.144
64 bytes from 10.0.0.144: icmp_seq=1 ttl=62 time=1.24 ms

```

# Load-balancing & Service Discovery - [Docs](https://docs.cilium.io/en/stable/network/clustermesh/services/)
```bash
#
cat << EOF | kubectl apply --context kind-west -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webpod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webpod
  template:
    metadata:
      labels:
        app: webpod
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - sample-app
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: webpod
        image: traefik/whoami
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: webpod
  labels:
    app: webpod
  annotations:
    service.cilium.io/global: "true"
spec:
  selector:
    app: webpod
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
EOF


#
cat << EOF | kubectl apply --context kind-east -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webpod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webpod
  template:
    metadata:
      labels:
        app: webpod
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - sample-app
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: webpod
        image: traefik/whoami
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: webpod
  labels:
    app: webpod
  annotations:
    service.cilium.io/global: "true"
spec:
  selector:
    app: webpod
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
EOF
```
```bash


# 확인
kwest get svc,ep webpod && keast get svc,ep webpod

kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
ID   Frontend               Service Type   Backend
...
13   10.2.37.24:80/TCP       ClusterIP      1 => 10.1.1.89:80/TCP (active)      
                                            2 => 10.0.1.27:80/TCP (active)      
                                            3 => 10.1.1.213:80/TCP (active)     
                                            4 => 10.0.1.19:80/TCP (active) 
                                            
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
ID   Frontend               Service Type   Backend
...
13   10.3.156.11:80/TCP     ClusterIP      1 => 10.1.1.89:80/TCP (active)      
                                           2 => 10.0.1.27:80/TCP (active)      
                                           3 => 10.1.1.213:80/TCP (active)     
                                           4 => 10.0.1.19:80/TCP (active) 
#
kubectl exec -it curl-pod --context kind-west -- ping -c 1 10.1.0.128
kubectl exec -it curl-pod --context kind-east -- ping -c 1 10.0.0.144

#
kubectl exec -it curl-pod --context kind-west -- sh -c 'while true; do curl -s --connect-timeout 1 webpod ; sleep 1; echo "---"; done;'
kubectl exec -it curl-pod --context kind-east -- sh -c 'while true; do curl -s --connect-timeout 1 webpod ; sleep 1; echo "---"; done;'

# 현재 Service Annotations 설정
kwest describe svc webpod | grep Annotations -A1
Annotations:              service.cilium.io/global: true
Selector:                 app=webpod

keast describe svc webpod | grep Annotations -A1
Annotations:              service.cilium.io/global: true
Selector:                 app=webpod


# 모니터링 : 반복 접속해두기
kubectl exec -it curl-pod --context kind-west -- sh -c 'while true; do curl -s --connect-timeout 1 webpod ; sleep 1; echo "---"; done;'


#
kwest scale deployment webpod --replicas 1
kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity

# 로컬 k8s 에 목적지가 없을 경우 어떻게 되나요?
kwest scale deployment webpod --replicas 0
kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity

#
kwest scale deployment webpod --replicas 2
kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity


# tcpdump 확인 : dataplane flow 확인
docker exec -it west-control-plane tcpdump -i any tcp port 80 -nnq 
docker exec -it west-worker tcpdump -i any tcp port 80 -nnq
docker exec -it east-control-plane tcpdump -i any tcp port 80 -nnq 
docker exec -it east-worker tcpdump -i any tcp port 80 -nnq

```

# Service Affinity - [Docs](https://docs.cilium.io/en/stable/network/clustermesh/affinity/)
![[Pasted image 20250810214509.png]]

```bash
# 모니터링 : 반복 접속해두기
kubectl exec -it curl-pod --context kind-west -- sh -c 'while true; do curl -s --connect-timeout 1 webpod ; sleep 1; echo "---"; done;'


# 현재 Service Annotations 설정
kwest describe svc webpod | grep Annotations -A1
keast describe svc webpod | grep Annotations -A1


# Session Affinity Local 설정
kwest annotate service webpod service.cilium.io/affinity=local --overwrite
kwest describe svc webpod | grep Annotations -A3

keast annotate service webpod service.cilium.io/affinity=local --overwrite
keast describe svc webpod | grep Annotations -A3

# 확인
kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
13   10.2.37.24:80/TCP       ClusterIP      1 => 10.1.1.89:80/TCP (active)               
                                            2 => 10.1.1.213:80/TCP (active)              
                                            3 => 10.0.1.64:80/TCP (active) (preferred)   
                                            4 => 10.0.1.69:80/TCP (active) (preferred) 

keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
13   10.3.156.11:80/TCP     ClusterIP      1 => 10.1.1.89:80/TCP (active) (preferred)    
                                           2 => 10.1.1.213:80/TCP (active) (preferred)   
                                           3 => 10.0.1.64:80/TCP (active)                
                                           4 => 10.0.1.69:80/TCP (active) 

# 호출 확인
kubectl exec -it curl-pod --context kind-west -- sh -c 'while true; do curl -s --connect-timeout 1 webpod ; sleep 1; echo "---"; done;'
kubectl exec -it curl-pod --context kind-east -- sh -c 'while true; do curl -s --connect-timeout 1 webpod ; sleep 1; echo "---"; done;'


# 현재 상태에서 replicas 변경 후 확인 : preferred 동작 이해하기!
kwest scale deployment webpod --replicas 0
kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
13   10.2.37.24:80/TCP       ClusterIP      1 => 10.1.1.89:80/TCP (active)      
                                            2 => 10.1.1.213:80/TCP (active)
# 다시 기본 설정값 적용
kwest scale deployment webpod --replicas 2


# tcpdump 확인
docker exec -it west-control-plane tcpdump -i any tcp port 80 -nnq 
docker exec -it west-worker tcpdump -i any tcp port 80 -nnq
docker exec -it east-control-plane tcpdump -i any tcp port 80 -nnq 
docker exec -it east-worker tcpdump -i any tcp port 80 -nnq

```

# service.cilium.io/affinity
```bash
# 현재 설정 상태 확인
kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
kwest describe svc webpod | grep Annotations -A3
keast describe svc webpod | grep Annotations -A3


# remote
kwest annotate service webpod service.cilium.io/affinity=remote --overwrite
keast annotate service webpod service.cilium.io/affinity=remote --overwrite
kwest describe svc webpod | grep Annotations -A3
keast describe svc webpod | grep Annotations -A3

kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity

# 호출 확인
kubectl exec -it curl-pod --context kind-west -- sh -c 'while true; do curl -s --connect-timeout 1 webpod ; sleep 1; echo "---"; done;'
kubectl exec -it curl-pod --context kind-east -- sh -c 'while true; do curl -s --connect-timeout 1 webpod ; sleep 1; echo "---"; done;'

#
kwest scale deployment webpod --replicas 0
kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity

# 다시 설정 원복
kwest scale deployment webpod --replicas 2

```

# service.cilium.io/shared
```bash
# 현재 설정 상태 확인
kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
kwest describe svc webpod | grep Annotations -A3
keast describe svc webpod | grep Annotations -A3


# local
kwest annotate service webpod service.cilium.io/affinity=local --overwrite
keast annotate service webpod service.cilium.io/affinity=local --overwrite
kwest describe svc webpod | grep Annotations -A3
keast describe svc webpod | grep Annotations -A3

kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity


# shared=false : 적용 시, 다른 k8s 클러스터가 영향을 받는다!
kwest annotate service webpod service.cilium.io/shared=false
kwest describe svc webpod | grep Annotations -A3

kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity

# 호출 확인
kubectl exec -it curl-pod --context kind-west -- sh -c 'while true; do curl -s --connect-timeout 1 webpod ; sleep 1; echo "---"; done;'
kubectl exec -it curl-pod --context kind-east -- sh -c 'while true; do curl -s --connect-timeout 1 webpod ; sleep 1; echo "---"; done;'


# 다시 설정 원복
kwest annotate service webpod service.cilium.io/shared=true --overwrite
kwest describe svc webpod | grep Annotations -A3
keast describe svc webpod | grep Annotations -A3

kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list --clustermesh-affinity

```

# clustermesh-apiserver 파드 정보 확인 : krew pexec 플러그인 활용 - [Github](https://github.com/ssup2/kpexec)

[https://docs.cilium.io/en/stable/operations/troubleshooting/#state-propagation](https://docs.cilium.io/en/stable/operations/troubleshooting/#state-propagation)

```bash
#
kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium node list 
Name                      IPv4 Address   Endpoint CIDR   IPv6 Address   Endpoint CIDR   Source
east/east-control-plane   172.18.0.5     10.1.0.0/24                                    clustermesh
east/east-worker          172.18.0.4     10.1.1.0/24                                    clustermesh
west/west-control-plane   172.18.0.2     10.0.0.0/24                                    custom-resource
west/west-worker          172.18.0.3     10.0.1.0/24                                    local

keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium node list            
Name                      IPv4 Address   Endpoint CIDR   IPv6 Address   Endpoint CIDR   Source
east/east-control-plane   172.18.0.5     10.1.0.0/24                                    custom-resource
east/east-worker          172.18.0.4     10.1.1.0/24                                    local
west/west-control-plane   172.18.0.2     10.0.0.0/24                                    clustermesh
west/west-worker          172.18.0.3     10.0.1.0/24                                    clustermesh

kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium identity list
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium identity list

kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium bpf ipcache list
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium bpf ipcache list
kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium map get cilium_ipcache
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium map get cilium_ipcache


kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list
keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium service list

kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium bpf lb list                      
10.2.127.79:443/TCP (0)     0.0.0.0:0 (2) (0) [ClusterIP, InternalLocal, non-routable] 

keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium bpf lb list                      
10.3.196.180:443/TCP (0)   0.0.0.0:0 (2) (0) [ClusterIP, InternalLocal, non-routable]


#
kubectl describe pod -n kube-system -l k8s-app=clustermesh-apiserver
...
Containers:
  etcd:
    Container ID:  containerd://7668abdd87c354ab9295ff9f0aa047b3bd658a8671016c4564a1045f9e32cb9f
    Image:         quay.io/cilium/clustermesh-apiserver:v1.17.6@sha256:f619e97432db427e1511bf91af3be8ded418c53a353a09629e04c5880659d1df
    Image ID:      quay.io/cilium/clustermesh-apiserver@sha256:f619e97432db427e1511bf91af3be8ded418c53a353a09629e04c5880659d1df
    Ports:         2379/TCP, 9963/TCP
    Host Ports:    0/TCP, 0/TCP
    Command:
      /usr/bin/etcd
    Args:
      --data-dir=/var/run/etcd
      --name=clustermesh-apiserver
      --client-cert-auth
      --trusted-ca-file=/var/lib/etcd-secrets/ca.crt
      --cert-file=/var/lib/etcd-secrets/tls.crt
      --key-file=/var/lib/etcd-secrets/tls.key
      --listen-client-urls=https://127.0.0.1:2379,https://[$(HOSTNAME_IP)]:2379
      --advertise-client-urls=https://[$(HOSTNAME_IP)]:2379
      --initial-cluster-token=$(INITIAL_CLUSTER_TOKEN)
      --auto-compaction-retention=1
      --listen-metrics-urls=http://[$(HOSTNAME_IP)]:9963
      --metrics=basic
  ...
  apiserver:
    Container ID:  containerd://96da1c1ca39cb54d9e4def9333195f9a3812bf4978adcbe72bf47202b029e28d
    Image:         quay.io/cilium/clustermesh-apiserver:v1.17.6@sha256:f619e97432db427e1511bf91af3be8ded418c53a353a09629e04c5880659d1df
    Image ID:      quay.io/cilium/clustermesh-apiserver@sha256:f619e97432db427e1511bf91af3be8ded418c53a353a09629e04c5880659d1df
    Ports:         9880/TCP, 9962/TCP
    Host Ports:    0/TCP, 0/TCP
    Command:
      /usr/bin/clustermesh-apiserver
    Args:
      clustermesh
      --debug
      --cluster-name=$(CLUSTER_NAME)
      --cluster-id=$(CLUSTER_ID)
      --kvstore-opt=etcd.config=/var/lib/cilium/etcd-config.yaml
      --kvstore-opt=etcd.qps=20
      --kvstore-opt=etcd.bootstrapQps=10000
      --max-connected-clusters=255
      --health-port=9880
      --enable-external-workloads=false
      --prometheus-serve-addr=:9962
      --controller-group-metrics=all

# etcd 컨터이너 로그 확인
kubectl logs -n kube-system deployment/clustermesh-apiserver -c etcd

# apiserver 컨터이너 로그 확인
kubectl logs -n kube-system deployment/clustermesh-apiserver -c apiserver


#
kwest exec -it -n kube-system ds/cilium -c cilium-agent -- cilium status --all-clusters
ClusterMesh:            1/1 remote clusters ready, 1 global-services
   west: ready, 2 nodes, 9 endpoints, 7 identities, 1 services, 0 MCS-API service exports, 0 reconnections (last: never)
   └  etcd: 1/1 connected, leases=0, lock leases=0, has-quorum=true: endpoint status checks are disabled, ID: da1e6d481fd94dd
   └  remote configuration: expected=true, retrieved=true, cluster-id=1, kvstoremesh=false, sync-canaries=true, service-exports=disabled
   └  synchronization status: nodes=true, endpoints=true, identities=true, services=true

keast exec -it -n kube-system ds/cilium -c cilium-agent -- cilium status --all-clusters
ClusterMesh:            1/1 remote clusters ready, 1 global-services
   west: ready, 2 nodes, 9 endpoints, 7 identities, 1 services, 0 MCS-API service exports, 0 reconnections (last: never)
   └  etcd: 1/1 connected, leases=0, lock leases=0, has-quorum=true: endpoint status checks are disabled, ID: da1e6d481fd94dd
   └  remote configuration: expected=true, retrieved=true, cluster-id=1, kvstoremesh=false, sync-canaries=true, service-exports=disabled
   └  synchronization status: nodes=true, endpoints=true, identities=true, services=true


# etcd 컨테이너 bash 진입 후 확인
# krew pexec 플러그인 활용 https://github.com/ssup2/kpexec
kubectl get pod -n kube-system -l k8s-app=clustermesh-apiserver
DPOD=clustermesh-apiserver-5cf45db9cc-zxhsd

kubectl pexec $DPOD -it -T -n kube-system -- bash
------------------------------------------------
ps -ef
ps -ef -T -o pid,ppid,comm,args
ps -ef -T -o args
COMMAND
/usr/bin/etcd --data-dir=/var/run/etcd --name=clustermesh-apiserver --client-cert-auth --trusted-ca-file=/var/lib/etcd-se

cat /proc/1/cmdline ; echo
/usr/bin/etcd--data-dir=/var/run/etcd--name=clustermesh-apiserver--client-cert-auth--trusted-ca-file=/var/lib/etcd-secrets/ca.crt--cert-file=/var/lib/etcd-secrets/tls.crt--key-file=/var/lib/etcd-secrets/tls.key--listen-client-urls=https://127.0.0.1:2379,https://[10.1.1.243]:2379--advertise-client-urls=https://[10.1.1.243]:2379--initial-cluster-token=c044b25f-dfd4-4eb5-b3ab-05dbfb60117c--auto-compaction-retention=1--listen-metrics-urls=http://[10.1.1.243]:9963--metrics=basic

ss -tnlp
ss -tnp
ESTAB     0          0                 10.1.1.243:2379             172.18.0.5:51880      users:(("etcd",pid=1,fd=15))     
ESTAB     0          0                 10.1.1.243:2379             172.18.0.5:59850      users:(("etcd",pid=1,fd=14)) 

exit
------------------------------------------------

kubectl pexec $DPOD -it -T -n kube-system -c apiserver -- bash
------------------------------------------------
ps -ef
ps -ef -T -o pid,ppid,comm,args
ps -ef -T -o args
cat /proc/1/cmdline ; echo
usr/bin/clustermesh-apiserverclustermesh--debug--cluster-name=east--cluster-id=2--kvstore-opt=etcd.config=/var/lib/cilium/etcd-config.yaml--kvstore-opt=etcd.qps=20--kvstore-opt=etcd.bootstrapQps=10000--max-connected-clusters=255--health-port=9880--enable-external-workloads=false--prometheus-serve-addr=:9962--controller-group-metrics=all

ss -tnlp
ss -tnp
ESTAB    0         0                127.0.0.1:42286          127.0.0.1:2379     users:(("clustermesh-api",pid=1,fd=7))       
ESTAB    0         0               10.1.1.243:38300         172.18.0.5:6443     users:(("clustermesh-api",pid=1,fd=6)) 

exit
------------------------------------------------

```


# 도전과제7` 만약 west 는 요청 클라이언트만 있고, east 는 대상 서버만 존재 시, ClusterMesh 서비스 통신 최적 설정을 샘플 App대상으로 해보기

^ca66e8

    - 대상 서버가 없는 west 에 Service(Global) 를 만들어야 되는지?
    - west 요청 클라이언트가 호출하는 주소(엔드포인트)는 어떤 주소가 되는지?

# 도전과제8 Cilium ClusterMesh 환경에서 NetworkPolicies : 실습 해보자 - [Docs](https://docs.cilium.io/en/stable/network/clustermesh/policy/)

^073695

# 도전과제9 Cilium DataStore(Kvstore) 를 설정해보고, Cilium 기본 CRD 모드와의 장단점 비교 정리 - [Docs](https://docs.cilium.io/en/stable/overview/component-overview/)

^0c9de0

```bash
#
cat << EOF > etcd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
  labels:
    app: etcd
spec:
  containers:
  - name: etcd
    image: quay.io/coreos/etcd:v3.5.0
    command:
      - /usr/local/bin/etcd
    args:
      - --name=etcd0
      - --data-dir=/etcd-data
      - --listen-client-urls=http://0.0.0.0:2379
      - --advertise-client-urls=http://etcd.kube-system.svc.cluster.local:2379
    ports:
      - containerPort: 2379
---
apiVersion: v1
kind: Service
metadata:
  name: etcd
  namespace: kube-system
spec:
  ports:
  - port: 2379
    targetPort: 2379
  selector:
    app: etcd      
EOF
```
```bash
kubectl apply -f etcd.yaml


#
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
--set etcd.enabled=true --set identityAllocationMode=kvstore --set "etcd.endpoints[0]=http://etcd.kube-system.svc.cluster.local:2379"

etcd:
  enabled: true
  endpoints:
  - http://etcd.kube-system.svc.cluster.local:2379

kubectl rollout restart deploy cilium-operator -n kube-system
kubectl rollout restart ds cilium -n kube-system

#
kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium status
KVStore:                 Ok   etcd: 1/1 connected, leases=1, lock leases=1, has-quorum=true: http://etcd.kube-system.svc.cluster.local:2379 - 3.5.0 (Leader)
...

```

# 도전과제10 Cilium ClusterMesh [EndpointSlice synchronization](https://docs.cilium.io/en/stable/network/clustermesh/services/#endpointslicesync) feature if you need Headless Services support.ㅋ

^d2933e

# 도전과제11 kind k8s 클러스터를 3개를 생성 후, 3개의 클러스터 모두를 ClusterMesh로 연결해보기

^da954e


```bash 
# 실습 후 kind 삭제 
kind delete cluster --name west && kind delete cluster --name east && docker rm -f mypc
```