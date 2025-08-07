IP Address Management : 네트워크 엔드포인트(컨테이너, 등)에 대한 IP 할당과 관리 - [Docs](https://docs.cilium.io/en/stable/network/concepts/ipam/)

| **Feature**                | **Kubernetes Host Scope** | **Cluster Scope (default)** | **Multi-Pool (Beta)** | **CRD-backed** | **AWS ENI…**   |
| -------------------------- | ------------------------- | --------------------------- | --------------------- | -------------- | -------------- |
| Tunnel routing             | ✅                         | ✅                           | ❌                     | ❌              | ❌              |
| Direct routing             | ✅                         | ✅                           | ✅                     | ✅              | ✅              |
| CIDR Configuration         | Kubernetes                | Cilium                      | Cilium                | External       | External (AWS) |
| Multiple CIDRs per cluster | ❌                         | ✅                           | ✅                     | N/A            | N/A            |
| Multiple CIDRs per node    | ❌                         | ❌                           | ✅                     | N/A            | N/A            |
| Dynamic CIDR/IP allocation | ❌                         | ❌                           | ✅                     | ✅              | ✅              |
|                            |                           |                             |                       |                |                |

>[!note]
기존 클러스터의 IPAM 모드를 변경하지 마세요. 
라이브 환경에서 IPAM 모드를 변경하면 기존 워크로드의 지속적인 연결 중단이 발생할 수 있습니다. 
IPAM 모드를 변경하는 가장 안전한 방법은 새로운 IPAM 구성으로 새로운 Kubernetes 클러스터를 설치하는 것입니다.


## Kubernetes Host Scope : - [Docs](https://docs.cilium.io/en/stable/network/concepts/ipam/kubernetes/)

^7af1b0

- Kubernetes 호스트 범위 IPAM 모드는 `ipam: Kubernetes`에서 활성화되며 클러스터의 각 개별 노드에 주소 할당을 위임합니다.
- IP는 Kubernetes에 의해 각 노드에 연결된 PodCIDR 범위에서 할당됩니다.
- 이 모드에서는 Cilium 에이전트가 `Kubernetes v1.Node` 객체를 통해 PodCIDR 범위가 다음 방법 중 하나를 통해 활성화된 모든 주소 패밀리에 대해 제공될 때까지 시작 시 대기합니다:
![[Pasted image 20250727231927.png]]

```bash
# 클러스터 정보 확인
kubectl cluster-info dump | grep -m 2 -E "cluster-cidr|service-cluster-ip-range"
	"--service-cluster-ip-range=10.96.0.0/16",
	"--cluster-cidr=10.244.0.0/16",

# ipam 모드 확인
cilium config view | grep ^ipam
ipam kubernetes

# 노드별 파드에 할당되는 IPAM(PodCIDR) 정보 확인
# --allocate-node-cidrs=true 로 설정된 kube-controller-manager에서 CIDR을 자동 할당함
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
k8s-ctr 10.244.0.0/24
k8s-w1  10.244.1.0/24

kc describe pod -n kube-system kube-controller-manager-k8s-ctr
...
    Command:
      kube-controller-manager
      --allocate-node-cidrs=true
      --cluster-cidr=10.244.0.0/16
      --service-cluster-ip-range=10.96.0.0/16
...

kubectl get ciliumnode -o json | grep podCIDRs -A2

# 파드 정보 : 상태, 파드 IP 확인
kubectl get ciliumendpoints.cilium.io -A

```

# Sample Application 배포
```bash
# 샘플 애플리케이션 배포
cat << EOF | kubectl apply -f -
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
spec:
  selector:
    app: webpod
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
EOF


# k8s-ctr 노드에 curl-pod 파드 배포
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: curl-pod
  labels:
    app: curl
spec:
  nodeName: k8s-ctr
  containers:
  - name: curl
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
EOF
```

# Sample Application 확인
```bash
# 배포 확인
kubectl get deploy,svc,ep webpod -owide
kubectl get endpointslices -l app=webpod
kubectl get ciliumendpoints # IP 확인
kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium-dbg endpoint list

# 통신 확인
kubectl exec -it curl-pod -- curl webpod | grep Hostname
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s webpod | grep Hostname; sleep 1; done'
```

```bash
# hubble ui 웹 접속 주소 확인 : default 네임스페이스 확인
NODEIP=$(ip -4 addr show eth1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
echo -e "http://$NODEIP:30003"

# hubble relay 포트 포워딩 실행
cilium hubble port-forward&
hubble status


# flow log 모니터링
hubble observe -f --protocol tcp --to-pod curl-pod
hubble observe -f --protocol tcp --from-pod curl-pod
hubble observe -f --protocol tcp --pod curl-pod
l 26 08:15:33.840: default/curl-pod (ID:37934) <> 10.96.88.194:80 (world) pre-xlate-fwd TRACED (TCP)
Jul 26 08:15:33.840: default/curl-pod (ID:37934) <> default/webpod-697b545f57-2h59t:80 (ID:23913) post-xlate-fwd TRANSLATED (TCP)
Jul 26 08:15:33.840: default/curl-pod:53092 (ID:37934) -> default/webpod-697b545f57-2h59t:80 (ID:23913) to-network FORWARDED (TCP Flags: SYN)
Jul 26 08:15:33.841: default/curl-pod:53092 (ID:37934) <- default/webpod-697b545f57-2h59t:80 (ID:23913) to-endpoint FORWARDED (TCP Flags: SYN, ACK)
pre-xlate-fwd , TRACED : NAT (IP 변환) 전 , 추적 중인 flow
post-xlate-fwd , TRANSLATED : NAT 후의 흐름 , NAT 변환이 일어났음

# 호출 시도
kubectl exec -it curl-pod -- curl webpod | grep Hostname
혹은
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s webpod | grep Hostname; sleep 1; done'


# tcpdump 확인 : 파드 IP 확인
tcpdump -i eth1 tcp port 80 -nn
17:23:25.920613 IP 10.244.0.144.39112 > 10.244.1.180.80: Flags [P.], seq 1:71, ack 1, win 502, options [nop,nop,TS val 3745105977 ecr 1971332111], length 70: HTTP: GET / HTTP/1.1

#
tcpdump -i eth1 tcp port 80 -w /tmp/http.pcap

#
termshark -r /tmp/http.pcap
```
![[Pasted image 20250727233312.png]]

# [Cilium] Cluster Scope & 마이그레이션 실습 - [Docs](https://docs.cilium.io/en/stable/network/concepts/ipam/cluster-pool/) , [IPAM](https://docs.cilium.io/en/stable/network/kubernetes/ipam-cluster-pool/)

^4a689c

![[Pasted image 20250727233549.png]]

- 클러스터 범위 IPAM 모드는 각 노드에 노드별 PodCIDR을 할당하고 각 노드에 호스트 범위 할당기를 사용하여 IP를 할당합니다.
- 따라서 이 모드는 Kubernetes 호스트 범위 모드와 유사합니다.
- 차이점은 Kubernetes가 `Kubernetes v1.Node` 리소스를 통해 노드별 PodCIDR을 할당하는 대신, Cilium 운영자가 `v2.CiliumNode` 리소스(CRD)를 통해 노드별 PodCIDR을 관리한다는 점입니다.
- 이 모드의 장점은 Kubernetes가 노드별 PodCIDR을 나눠주도록 구성되는 것에 의존하지 않는다는 점입니다.
- 최소 마스크 길이는 /30이며 권장 최소 마스크 길이는 /29 이상입니다. 2개 주소는 예약됨(네트워크, 브로드캐스트 주소)
- `10.0.0.0/8` is the default pod CIDR. `clusterPoolIPv4PodCIDRList`

```bash
# 반복 요청 해두기
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s webpod | grep Hostname; sleep 1; done'


# Cluster Scopre 로 설정 변경
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
--set ipam.mode="cluster-pool" --set ipam.operator.clusterPoolIPv4PodCIDRList={"172.20.0.0/16"} --set ipv4NativeRoutingCIDR=172.20.0.0/16

kubectl -n kube-system rollout restart deploy/cilium-operator # 오퍼레이터 재시작 필요
kubectl -n kube-system rollout restart ds/cilium


# 변경 확인
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
cilium config view | grep ^ipam
ipam   cluster-pool

kubectl get ciliumnode -o json | grep podCIDRs -A2
kubectl get ciliumendpoints.cilium.io -A


# 
kubectl delete ciliumnode k8s-w1
kubectl -n kube-system rollout restart ds/cilium
kubectl get ciliumnode -o json | grep podCIDRs -A2
kubectl get ciliumendpoints.cilium.io -A

#
kubectl delete ciliumnode k8s-ctr
kubectl -n kube-system rollout restart ds/cilium
kubectl get ciliumnode -o json | grep podCIDRs -A2
kubectl get ciliumendpoints.cilium.io -A # 파드 IP 변경 되는가?

# 노드의 poccidr static routing 자동 변경 적용 확인
ip -c route
sshpass -p 'vagrant' ssh vagrant@k8s-w1 ip -c route


# 직접 rollout restart 하자! 
kubectl get pod -A -owide | grep 10.244.

kubectl -n kube-system rollout restart deploy/hubble-relay deploy/hubble-ui
kubectl -n cilium-monitoring rollout restart deploy/prometheus deploy/grafana
kubectl rollout restart deploy/webpod
kubectl delete pod curl-pod

#
cilium hubble port-forward&


# k8s-ctr 노드에 curl-pod 파드 배포
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: curl-pod
  labels:
    app: curl
spec:
  nodeName: k8s-ctr
  containers:
  - name: curl
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
EOF

# 파드 IP 변경 확인!
kubectl get ciliumendpoints.cilium.io -A

# 반복 요청
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s webpod | grep Hostname; sleep 1; done'

```

# [Cilium CNI Chaining] AWS VPC CNI plugin - [Docs](https://docs.cilium.io/en/stable/installation/cni-chaining-aws-cni/)

^41490d
![[Pasted image 20250727234658.png]]

- 이 가이드는 Cilium을 AWS VPC CNI 플러그인과 결합하여 설정하는 방법을 설명합니다.
- 이 하이브리드 모드에서는 AWS VPC CNI 플러그인이 가상 네트워크 장치 설정뿐만 아니라 ENI를 통한 IP 주소 관리(IPAM)도 담당합니다.
- 주어진 포드에 대해 초기 네트워킹이 설정된 후, Cilium CNI 플러그인은 네트워크 정책을 시행하고 로드 밸런싱을 수행하며 암호화를 제공하기 위해 AWS VPC CNI 플러그인이 설정한 네트워크 장치에 eBPF 프로그램을 연결하도록 호출됩니다.

## AWS CNI 
![[Pasted image 20250727234816.png]]
- AWS-CNI 역할 : Device plumbing, IPAM(ENI), Routing(Native-Routing 등)
- Cilium 역할 : LB, Network Policy, Encrption, Multi-Cluster, Visiblity
- 설정
```bash
helm install cilium cilium/cilium --version 1.17.6 \
  --namespace kube-system \
  --set cni.chainingMode=aws-cni \
  --set cni.exclusive=false \
  --set enableIPv4Masquerade=false \
  --set routingMode=native
```

## AWS ENI IPAM 모드 - [Docs](https://docs.cilium.io/en/stable/network/concepts/ipam/eni/)
![[Pasted image 20250727235013.png]]
- AWS ENI 할당기는 AWS 클라우드에서 실행되는 Cilium 배포에 특화되어 있으며, AWS EC2 API와 통신하여 AWS Elastic Network Interface(ENI)의 IP를 기반으로 IP 할당을 수행합니다.
- 이 아키텍처는 대규모 클러스터에서 속도 제한 문제를 방지하기 위해 단일 운영자만 EC2 서비스 API와 통신할 수 있도록 보장합니다.
- 사전 할당 워터마크는 클러스터에서 새 포드가 예약될 때 EC2 API에 연락할 필요 없이 노드에서 항상 사용할 수 있도록 여러 IP 주소를 유지하는 데 사용됩니다.
- 설정
```bash
helm repo add cilium https://helm.cilium.io/
helm upgrade --install cilium cilium/cilium \
  --namespace kube-system \
  --set eni.enabled=true \
  --set ipam.mode=eni \
  --set egressMasqueradeInterfaces=eth0 \
  --set routingMode=native \
  --set kubeProxyReplacement=true \
  --set operator.replicas=1

```
•	eni.enabled=true : ENI 모드 활성화
•	ipam.mode=eni : IP 관리 방식으로 ENI 사용 지정
•	egressMasqueradeInterfaces=eth0 : NAT 수행할 인터페이스 지정(필요시 VPC 환경 맞게 변경)
•	routingMode=native : 네이티브 라우팅 활성화(터널 비활성화)
•	kubeProxyReplacement=true : kube-proxy 대체(필수는 아님, 운영환경 따라 설정)
•	operator.replicas=1 : cilium-operator 프로세스 개수

### 워터마크(ENI IP 할당 관련 사전 할당 수) 조정
ENI IPAM 워터마크(사전 할당 IP, 최소/최대 등)는 Helm에서 아래와 같이 세팅 가능합니다:
```bash 
helm upgrade --install cilium cilium/cilium \
  ...기본값...
  --set eni.preAllocate=8 \
  --set eni.minAllocate=8 \
  --set eni.maxAllocate=20 \
  --set eni.maxAboveWatermark=2
```
•	eni.preAllocate: 미리 확보해두는 IP 수(새로운 Pod 예약시 대기시간 최소화)
•	eni.minAllocate: 항상 최소 보유할 IP 수
•	eni.maxAllocate: 노드당 최대 IP 수(초과 요청 생기면 할당 안함)
•	eni.maxAboveWatermark: 여유분 IP만큼만 초과 보유 허용

## 결론
Cilium ENI IPAM을 사용하면 CNI Chaining이 필요하지 않으며, Cilium이 ENI와 직접 연동(IP 할당 포함)하는 구조입니다.
•	CNI Chaining 방식은 aws-vpc-cni(ENI) + Cilium처럼 두 개의 CNI가 역할을 나눌 때 활용되며, 최신 Cilium ENI 운용에서는 주로 단독 운용이 권장됩니다.

