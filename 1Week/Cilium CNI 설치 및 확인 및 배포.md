# Cilium 시스템 요건 

- Cilium 시스템 요구 사항 확인 - [Docs](https://docs.cilium.io/en/stable/operations/system_requirements/)
    - AMD64 또는 AArch64 CPU 아키텍처를 사용하는 호스트
    - [Linux 커널](https://docs.cilium.io/en/stable/operations/system_requirements/#linux-kernel) 5.4 이상 또는 동등 버전(예: RHEL 8.6의 경우 4.18)
```bash
#
arch
aarch64

#
uname -r
6.8.0-53-generic
```

- 커널 구성 옵션 활성화
    
    ```bash
    # [커널 구성 옵션] 기본 요구 사항 
    grep -E 'CONFIG_BPF|CONFIG_BPF_SYSCALL|CONFIG_NET_CLS_BPF|CONFIG_BPF_JIT|CONFIG_NET_CLS_ACT|CONFIG_NET_SCH_INGRESS|CONFIG_CRYPTO_SHA1|CONFIG_CRYPTO_USER_API_HASH|CONFIG_CGROUPS|CONFIG_CGROUP_BPF|CONFIG_PERF_EVENTS|CONFIG_SCHEDSTATS' /boot/config-$(uname -r)
    CONFIG_BPF=y
    CONFIG_BPF_SYSCALL=y
    CONFIG_BPF_JIT=y
    CONFIG_NET_CLS_BPF=m
    CONFIG_NET_CLS_ACT=y
    CONFIG_NET_SCH_INGRESS=m
    CONFIG_CRYPTO_SHA1=y
    CONFIG_CRYPTO_USER_API_HASH=m
    CONFIG_CGROUPS=y
    CONFIG_CGROUP_BPF=y
    CONFIG_PERF_EVENTS=y
    CONFIG_SCHEDSTATS=y
    
    # [커널 구성 옵션] Requirements for Tunneling and Routing
    grep -E 'CONFIG_VXLAN=y|CONFIG_VXLAN=m|CONFIG_GENEVE=y|CONFIG_GENEVE=m|CONFIG_FIB_RULES=y' /boot/config-$(uname -r)
    CONFIG_FIB_RULES=y # 커널에 내장됨
    CONFIG_VXLAN=m # 모듈로 컴파일됨 → 커널에 로드해서 사용
    CONFIG_GENEVE=m # 모듈로 컴파일됨 → 커널에 로드해서 사용
    
    ## (참고) 커널 로드
    lsmod | grep -E 'vxlan|geneve'
    modprobe geneve
    lsmod | grep -E 'vxlan|geneve'
    
    # [커널 구성 옵션] Requirements for L7 and FQDN Policies
    grep -E 'CONFIG_NETFILTER_XT_TARGET_TPROXY|CONFIG_NETFILTER_XT_TARGET_MARK|CONFIG_NETFILTER_XT_TARGET_CT|CONFIG_NETFILTER_XT_MATCH_MARK|CONFIG_NETFILTER_XT_MATCH_SOCKET' /boot/config-$(uname -r)
    CONFIG_NETFILTER_XT_TARGET_CT=m
    CONFIG_NETFILTER_XT_TARGET_MARK=m
    CONFIG_NETFILTER_XT_TARGET_TPROXY=m
    CONFIG_NETFILTER_XT_MATCH_MARK=m
    CONFIG_NETFILTER_XT_MATCH_SOCKET=m
    
    ...
    
    # [커널 구성 옵션] Requirements for Netkit Device Mode
    grep -E 'CONFIG_NETKIT=y|CONFIG_NETKIT=m' /boot/config-$(uname -r)
    
    ```

# 고급 기능 동작을 위한 최소 커널 버전

|Cilium Feature|Minimum Kernel Version|
|---|---|
|[WireGuard Transparent Encryption](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/#encryption-wg)|>= 5.6|
|Full support for [Session Affinity](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#session-affinity)|>= 5.7|
|BPF-based proxy redirection|>= 5.7|
|Socket-level LB bypass in pod netns|>= 5.7|
|L3 devices|>= 5.8|
|BPF-based host routing|>= 5.10|
|[Multicast Support in Cilium (Beta)](https://docs.cilium.io/en/stable/network/multicast/#enable-multicast) (AMD64)|>= 5.10|
|IPv6 BIG TCP support|>= 5.19|
|[Multicast Support in Cilium (Beta)](https://docs.cilium.io/en/stable/network/multicast/#enable-multicast) (AArch64)|>= 6.0|
|IPv4 BIG TCP support|>= 6.3|
Cilium 동작(Node 간)을 위한 방화벽 규칙 : 해당 포트 인/아웃 허용 필요 - [Docs](https://docs.cilium.io/en/stable/operations/system_requirements/#firewall-rules)

Mounted eBPF filesystem : 일부 배포판 마운트되어 있음, 혹은 Cilium 설치 시 마운트 시도 - [Docs](https://docs.cilium.io/en/stable/operations/system_requirements/#mounted-ebpf-filesystem)
```bash
#
mount | grep /sys/fs/bpf
bpf on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)
```

Privileges : Cilium 동작을 위해서 관리자 수준 권한 필요
- Cilium interacts with the Linux kernel to install eBPF program which will then perform networking tasks and implement security rules. In order to install eBPF programs system-wide, `CAP_SYS_ADMIN` privileges are required. These privileges must be granted to `cilium-agent`.
    
    The quickest way to meet the requirement is to run `cilium-agent` as root and/or as privileged container.
    
- Cilium requires access to the host networking namespace. For this purpose, the Cilium pod is scheduled to run in the host networking namespace directly.

------
# 도전과제
Cilium 시스템 요구 사항을 점검하는 ‘Bash Script 나 Ansible Playbook’ 를 만들어서 적용해보기
```
#!/bin/bash
grep BPF /boot/config-$(uname -r)
grep VXLAN /boot/config-$(uname -r)
mount | grep /sys/fs/bpf
uname -m
uname -r

```
모든 노드에 배포/실행 후 값 확인
	•	정상 값: 필수 커널 옵션=Y 또는 =m, bpf 마운트, kernel version, arch 등

-------

# Cilium 설치 Without kube-proxy* - [Docs](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)

## 기존 Flannel CNI 제거
   
```bash
#
helm uninstall -n kube-flannel flannel
helm list -A

#
kubectl get all -n kube-flannel
kubectl delete ns kube-flannel

#
kubectl get pod -A -owide
```

## k8s-ctr, k8s-w1, k8s-w2 모든 노드에서 아래 실행

```bash
# 제거 전 확인
ip -c link
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i ip -c link ; echo; done

brctl show
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i brctl show ; echo; done

ip -c route
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i ip -c route ; echo; done


# vnic 제거
ip link del flannel.1
ip link del cni0

for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i sudo ip link del flannel.1 ; echo; done
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i sudo ip link del cni0 ; echo; done

# 제거 확인
ip -c link
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i ip -c link ; echo; done

brctl show
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i brctl show ; echo; done

ip -c route
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i ip -c route ; echo; done
```
## 기존 kube-proxy 제거
```bash
#
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy

# 배포된 파드의 IP는 남겨져 있음
kubectl get pod -A -owide

#
kubectl exec -it curl-pod -- curl webpod

#
iptables-save
```

## k8s-ctr, k8s-w1, k8s-w2 모든 노드에서 아래 실행
```bash
# Run on each node with root permissions:
iptables-save | grep -v KUBE | grep -v FLANNEL | iptables-restore
iptables-save

sshpass -p 'vagrant' ssh vagrant@k8s-w1 "sudo iptables-save | grep -v KUBE | grep -v FLANNEL | sudo iptables-restore"
sshpass -p 'vagrant' ssh vagrant@k8s-w1 sudo iptables-save

sshpass -p 'vagrant' ssh vagrant@k8s-w2 "sudo iptables-save | grep -v KUBE | grep -v FLANNEL | sudo iptables-restore"
sshpass -p 'vagrant' ssh vagrant@k8s-w2 sudo iptables-save

#
kubectl get pod -owide
```

## 노드별 파드에 할당되는 IPAM(PodCIDR) 정보 확인
```bash
#--allocate-node-cidrs=true 로 설정된 kube-controller-manager에서 CIDR을 자동 할당함
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
k8s-ctr 10.244.0.0/24
k8s-w1  10.244.1.0/24
k8s-w2  10.244.2.0/24

kubectl get pod -owide

#
kc describe pod -n kube-system kube-controller-manager-k8s-ctr
...
    Command:
      kube-controller-manager
      --allocate-node-cidrs=true
      --cluster-cidr=10.244.0.0/16
      --service-cluster-ip-range=10.96.0.0/16
...
```

# Cilium 1.17.5 설치 with Helm
![[Pasted image 20250720022904.png]]
```bash
# Cilium 설치 with Helm
helm repo add cilium https://helm.cilium.io/

# 모든 NIC 지정 + bpf.masq=true + NoIptablesRules
helm install cilium cilium/cilium --version 1.17.5 --namespace kube-system \
--set k8sServiceHost=192.168.10.100 --set k8sServicePort=6443 \
--set kubeProxyReplacement=true \
--set routingMode=native \
--set autoDirectNodeRoutes=true \
--set ipam.mode="cluster-pool" \
--set ipam.operator.clusterPoolIPv4PodCIDRList={"172.20.0.0/16"} \
--set ipv4NativeRoutingCIDR=172.20.0.0/16 \
--set endpointRoutes.enabled=true \
--set installNoConntrackIptablesRules=true \
--set bpf.masquerade=true \
--set ipv6.enabled=false

# 확인
helm get values cilium -n kube-system
helm list -A
kubectl get crd
watch -d kubectl get pod -A

kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium-dbg status --verbose
KubeProxyReplacement:   True   [eth0    10.0.2.15 fd17:625c:f037:2:a00:27ff:fe71:19d8 fe80::a00:27ff:fe71:19d8, eth1   192.168.10.102 fe80::a00:27ff:fed3:64b (Direct Routing)]
Routing:                Network: Native   Host: BPF
Masquerading:           BPF   [eth0, eth1]   172.20.0.0/16 [IPv4: Enabled, IPv6: Disabled]
...

# 노드에 iptables 확인
iptables -t nat -S
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo iptables -t nat -S ; echo; done

iptables-save
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo iptables-save ; echo; done

```

## PodCIDR IPAM 확인
```bash
#
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
k8s-ctr 10.244.0.0/24
k8s-w1  10.244.1.0/24
k8s-w2  10.244.2.0/24

# 파드 IP 확인
kubectl get pod -owide
NAME                      READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE   READINESS GATES
curl-pod                  1/1     Running   0          22m   10.244.0.2   k8s-ctr   <none>           <none>
webpod-697b545f57-hbkmr   1/1     Running   0          22m   10.244.2.4   k8s-w2    <none>           <none>
webpod-697b545f57-vx27c   1/1     Running   0          22m   10.244.1.2   k8s-w1    <none>           <none>

#
kubectl get ciliumnodes
kubectl get ciliumnodes -o json | grep podCIDRs -A2
                    "podCIDRs": [
                        "172.20.0.0/24"
--
                    "podCIDRs": [
                        "172.20.1.0/24"
--
                    "podCIDRs": [

#
kubectl rollout restart deployment webpod
kubectl get pod -owide

# k8s-ctr 노드에 curl-pod 파드 배포
kubectl delete pod curl-pod --grace-period=0

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

kubectl get pod -owide
kubectl get ciliumendpoints
kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium-dbg endpoint list

# 통신 확인
kubectl exec -it curl-pod -- curl webpod | grep Hostname
kubectl exec -it curl-pod -- curl webpod | grep Hostname
```

## Cilium 설치 확인
```bash
# cilium cli 설치
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz >/dev/null 2>&1
tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz

# cilium 상태 확인
which cilium
cilium status
cilium config view
kubectl get cm -n kube-system cilium-config -o json | jq

#
cilium config set debug true && watch kubectl get pod -A
cilium config view | grep -i debug


# cilium daemon = cilium-dbg
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg config
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg status --verbose
...
KubeProxyReplacement:   True   [eth0    10.0.2.15 fd17:625c:f037:2:a00:27ff:fe71:19d8 fe80::a00:27ff:fe71:19d8, eth1   192.168.10.102 fe80::a00:27ff:fef6:fcbc (Direct Routing)]
Routing:                Network: Native   Host: BPF
Attach Mode:            TCX
Device Mode:            veth
Masquerading:           BPF   [eth0, eth1]   172.20.0.0/16 [IPv4: Enabled, IPv6: Disabled]
...
KubeProxyReplacement Details:
  Status:                 True
  Socket LB:              Enabled
  Socket LB Tracing:      Enabled
  Socket LB Coverage:     Full
  Devices:                eth0    10.0.2.15 fd17:625c:f037:2:a00:27ff:fe71:19d8 fe80::a00:27ff:fe71:19d8, eth1   192.168.10.102 fe80::a00:27ff:fef6:fcbc (Direct Routing)
  Mode:                   SNAT
  Backend Selection:      Random
  Session Affinity:       Enabled
  Graceful Termination:   Enabled
  NAT46/64 Support:       Disabled
  XDP Acceleration:       Disabled
  Services:
  - ClusterIP:      Enabled
  - NodePort:       Enabled (Range: 30000-32767) 
  - LoadBalancer:   Enabled 
  - externalIPs:    Enabled 
  - HostPort:       Enabled
...
```
![[Pasted image 20250720023138.png]]

![[Pasted image 20250720023153.png]]

## Routing
```bash
# Native-Routing + autoDirectNodeRoutes=true
ip -c route | grep 172.20 | grep eth1
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i ip -c route | grep 172.20 | grep eth1 ; echo; done

# hostNetwork 를 사용하지 않는 파드의 경우 endpointRoutes.enabled=true 설정으로 lxcY 인터페이스 생성됨
kubectl get ciliumendpoints -A
ip -c route | grep lxc
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i ip -c route | grep lxc ; echo; done
```

# Cilium CMD Cheatsheet
![[Isovalent - Cilium Cheat Sheet.pdf]]

```bash
# cilium 파드 이름
export CILIUMPOD0=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-ctr -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD1=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w1  -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD2=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w2  -o jsonpath='{.items[0].metadata.name}')
echo $CILIUMPOD0 $CILIUMPOD1 $CILIUMPOD2

# 단축키(alias) 지정
alias c0="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- cilium"
alias c1="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- cilium"
alias c2="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- cilium"

alias c0bpf="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- bpftool"
alias c1bpf="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- bpftool"
alias c2bpf="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- bpftool"


# endpoint
c0 endpoint list
c0 endpoint list -o json
c1 endpoint list
c2 endpoint list

c1 endpoint get <id>
c1 endpoint log <id>

## Enable debugging output on the cilium-dbg monitor for this endpoint
c1 endpoint config <id> Debug=true


# monitor
c1 monitor
c1 monitor -v
c1 monitor -v -v

## Filter for only the events related to endpoint
c1 monitor --related-to=<id>

## Show notifications only for dropped packet events
c1 monitor --type drop

## Don’t dissect packet payload, display payload in hex information
c1 monitor -v -v --hex

## Layer7
c1 monitor -v --type l7


# Manage IP addresses and associated information - IP List
c0 ip list

# IDENTITY :  1(host), 2(world), 4(health), 6(remote), 파드마다 개별 ID
c0 ip list -n

# Retrieve information about an identity
c0 identity list

# 엔드포인트 기준 ID
c0 identity list --endpoints

# 엔드포인트 설정 확인 및 변경
c0 endpoint config <엔트포인트ID>

# 엔드포인트 상세 정보 확인
c0 endpoint get <엔트포인트ID>

# 엔드포인트 로그 확인
c0 endpoint log <엔트포인트ID>

# Show bpf filesystem mount details
c0 bpf fs show

# bfp 마운트 폴더 확인
tree /sys/fs/bpf


# Get list of loadbalancer services
c0 service list
c1 service list
c2 service list

## Or you can get the loadbalancer information using bpf list
c0 bpf lb list
c1 bpf lb list
c2 bpf lb list

## List reverse NAT entries
c1 bpf lb list --revnat
c2 bpf lb list --revnat


# List connection tracking entries
c0 bpf ct list global
c1 bpf ct list global
c2 bpf ct list global

# Flush connection tracking entries
c0 bpf ct flush
c1 bpf ct flush
c2 bpf ct flush


# List all NAT mapping entries
c0 bpf nat list
c1 bpf nat list
c2 bpf nat list

# Flush all NAT mapping entries
c0 bpf nat flush
c1 bpf nat flush
c2 bpf nat flush

# Manage the IPCache mappings for IP/CIDR <-> Identity
c0 bpf ipcache list# Display cgroup metadata maintained by Cilium
c0 cgroups list
c1 cgroups list
c2 cgroups list


# List all open BPF maps
c0 map list
c1 map list --verbose
c2 map list --verbose

c1 map events cilium_lb4_services_v2
c1 map events cilium_lb4_reverse_nat
c1 map events cilium_lxc
c1 map events cilium_ipcache


# List all metrics
c1 metrics list


# List contents of a policy BPF map : Dump all policy maps
c0 bpf policy get --all
c1 bpf policy get --all -n
c2 bpf policy get --all -n


# Dump StateDB contents as JSON
c0 statedb dump


#
c0 shell -- db/show devices
c1 shell -- db/show devices
c2 shell -- db/show devices
```

# 통신 확인
## 노드간 파트 <-> 파드 통신 확인 
### Cilium 정보 확인
```bash
# 엔드포인트 정보 확인
kubectl get pod -owide
kubectl get svc,ep webpod
WEBPOD1IP=172.20.0.150

# BPF maps : 목적지 파드와 통신 시 어느곳으로 보내야 될지 확인할 수 있다
c0 map get cilium_ipcache
c0 map get cilium_ipcache | grep $WEBPOD1IP

# curl-pod 의 LXC 변수 지정
LXC=<k8s-ctr의 가장 나중에 lxc 이름>
LXC=*lxc8e4cb6673409*

# Node’s eBPF programs
## list of eBPF programs
c0bpf net show
c0bpf net show | grep $LXC 
*lxc096bf95a0ae0(14) tcx/ingress cil_from_container prog_id 1595 link_id 24 
lxc096bf95a0ae0(14) tcx/egress cil_to_container prog_id 1584 link_id 25* 

## Use bpftool prog show id to view additional information about a program, including a list of attached eBPF maps:
c0bpf prog show id <출력된 prog id 입력>
c0bpf prog show id *1584*
*1584: sched_cls  name cil_to_container  tag 0b3125767ba1861c  gpl
        loaded_at 2025-07-13T04:58:30+0000  uid 0
        xlated 1448B  jited 1144B  memlock 4096B  map_ids 242,62,243
        btf_id 464*

c0bpf map list
*...
62: percpu_hash  name cilium_metrics  flags 0x1
        key 8B  value 16B  max_entries 1024  memlock 19024B
...
242: array  name .rodata.config  flags 0x480
        key 4B  value 64B  max_entries 1  memlock 8192B
        btf_id 454  frozen
243: prog_array  name cilium_calls_00  flags 0x0
        key 4B  value 4B  max_entries 50  memlock 720B
        owner_prog_type sched_cls  owner jited
...*

### 다른 노드 간 ‘파드 → 파드’ 통신 확인
```bash
# vagrant ssh k8s-w1 , # vagrant ssh k8s-w2 각각 터미널 접속 후 아래 실행
ngrep -tW byline -d eth1 '' 'tcp port 80'

# [k8s-ctr] curl-pod 에서 curl 요청 시도
kubectl exec -it curl-pod -- curl $WEBPOD1IP

# 각각 터미널에서 출력 확인 : 파드의 소스 IP와 목적지 IP가 다른 노드의 서버 NIC에서 확인! : Native-Routung 
####
T 2025/07/13 14:57:40.848384 172.20.2.161:59448 -> 172.20.0.61:80 [AP] #4
GET / HTTP/1.1.
...
##
T 2025/07/13 14:57:40.851698 172.20.0.61:80 -> 172.20.2.161:59448 [AP] #6
HTTP/1.1 200 OK.
...
```


### 다른 노드 간 ‘파드 → 서비스(ClusterIP)’ 통신 확인* [https://velog.io/@haruband/K8SCilium-Socket-Based-LoadBalancing-기법](https://velog.io/@haruband/K8SCilium-Socket-Based-LoadBalancing-%EA%B8%B0%EB%B2%95)
   - 그림 왼쪽(네트워크 기반 로드밸런싱) vs 오른쪽(소켓 기반 로드밸런싱)

![[Pasted image 20250713224624.png]]

- Pod1 안에서 동작하는 앱이 connect() 시스템콜을 이용하여 소켓을 연결할 때 목적지 주소가 서비스 주소(10.10.8.55)이면 소켓의 목적지 주소를 바로 백엔드 주소(10.0.0.31)로 설정한다.
- 이후 앱에서 해당 소켓을 통해 보내는 모든 패킷의 목적지 주소는 이미 백엔드 주소(10.0.0.31)로 설정되어 있기 때문에 중간에 DNAT 변환 및 역변환 과정이 필요없어진다.
- destination NAT translation happens at the syscall level, before the packet is even built by the kernel.
- Socket operations : BPF socket operations program 은 root cgroup 에 연결되며 TCP event(ESTABLISHED) 에서 실행한다.
- Socket send/recv : The socket send/recv hook 은 TCP socket 의 모든 송수신 작업에서 실행, hook 에서 검사/삭제/리다이렉션을 할 수 있다

![[Pasted image 20250713224702.png]]

### 파드 네임스페이스에서 Socket-Based LoadBalancing 기법
![[Pasted image 20250713224728.png]]

- connect() 와 sendto() 소켓 함수에 연결된 프로그램(connect4, sendmsg4)에서는 소켓의 목적지 주소를 백엔드 주소와 포트로 변환하고, cilium_lb4_backends 맵에 백엔드 주소와 포트를 등록해놓는다.
- 이후 recvmsg() 소켓 함수에 연결된 프로그램(recvmsg4)에서는 cilium_lb4_reverse_nat 맵을 이용해서 목적지 주소와 포트를 다시 서비스 주소와 포트로 변환함

![[Pasted image 20250713224801.png]]

# 실습 확인
```bash
# curl 호출
kubectl exec -it curl-pod -- curl webpod

# 신규 터미널 : 파드에서 SVC(ClusterIP) 접속 시 tcpdump 로 확인 : ClusterIP가 소켓 레벨에서 이미 Endpoint 로 변경되었음을 확인!
kubectl exec curl-pod -- tcpdump -enni any -q
06:19:43.214116 eth0  Out ifindex 17 0e:0e:1f:0b:b4:70 172.20.2.70.33950 > 172.20.1.25.80: tcp 0
06:19:43.215346 eth0  In  ifindex 17 6a:20:d8:21:a8:1e 172.20.1.25.80 > 172.20.2.70.33950: tcp 0


# Socket-Based LoadBalancing 관련 설정들 확인
c0 status --verbose
...
KubeProxyReplacement Details:
  Status:                 True
  Socket LB:              Enabled
  Socket LB Tracing:      Enabled
  Socket LB Coverage:     Full
  Devices:                eth0    10.0.2.15 fd17:625c:f037:2:a00:27ff:fe71:19d8 fe80::a00:27ff:fe71:19d8, eth1   192.168.10.100 fe80::a00:27ff:fe5d:6cd7 (Direct Routing)
  Mode:                   SNAT
  Backend Selection:      Random
  Session Affinity:       Enabled
  Graceful Termination:   Enabled
  NAT46/64 Support:       Disabled
  XDP Acceleration:       Disabled
  Services:
  - ClusterIP:      Enabled
  - NodePort:       Enabled (Range: 30000-32767) 
  - LoadBalancer:   Enabled .
  
# syacall 호출 확인
kubectl exec curl-pod -- strace -c curl -s webpod
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 35.15    0.001112         370         3           sendto
 21.33    0.000675         225         3         1 connect
 19.09    0.000604         100         6         3 recvfrom
  6.67    0.000211           9        22           close
  4.01    0.000127           3        35           munmap
  3.57    0.000113           2        47        30 openat
  2.24    0.000071           7         9           ppoll
  2.15    0.000068          13         5           getsockname
  1.26    0.000040           0        63           mmap
  0.82    0.000026           0        28           rt_sigaction
  0.66    0.000021           0        27           read
  0.63    0.000020           5         4           socket
  0.38    0.000012           0        14           rt_sigprocmask
  0.32    0.000010          10         1           writev
  0.28    0.000009           0        10           lseek
  0.28    0.000009           3         3           readv
  0.25    0.000008           1         5           setsockopt
  0.22    0.000007           0        24           fcntl
  0.16    0.000005           5         1           newfstatat
  0.16    0.000005           5         1           eventfd2
  0.13    0.000004           1         3         3 ioctl
  0.06    0.000002           2         1           getsockopt
  0.06    0.000002           0         4           brk
  0.06    0.000002           2         1           getrandom
  0.03    0.000001           0         2           geteuid
  0.00    0.000000           0        12           fstat
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         1           getuid
  0.00    0.000000           0         1           getgid
  0.00    0.000000           0         1           getegid
  0.00    0.000000           0         1           execve
  0.00    0.000000           0        14           mprotect
------ ----------- ----------- --------- --------- ----------------
100.00    0.003164           8       353        37 total

# 상세 출력
kubectl exec curl-pod -- strace -s 65535 -f -tt curl -s webpod
...

# 특정 이벤트 필터링 : -e
## connect 로 출력되는 10.96.165.83 는 webpod Service 의 ClusterIP
kubectl exec curl-pod -- strace -e trace=connect     curl -s webpod
connect(5, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("10.96.165.83")}, 16) = 0
...

## connect 로 출력되는 172.20.2.70 는 curl-pod 의 파드 IP임. -> 목적지 webpod 파드 IP가 아님!
kubectl exec curl-pod -- strace -e trace=getsockname curl -s webpod
getsockname(4, {sa_family=AF_INET, sin_port=htons(48314), sin_addr=inet_addr("172.20.2.70")}, [128 => 16]) = 0
getsockname(5, {sa_family=AF_INET, sin_port=htons(39491), sin_addr=inet_addr("172.20.2.70")}, [16]) = 0
getsockname(4, {sa_family=AF_INET, sin_port=htons(53362), sin_addr=inet_addr("172.20.2.70")}, [128 => 16]) = 0
getsockname(4, {sa_family=AF_INET, sin_port=htons(53362), sin_addr=inet_addr("172.20.2.70")}, [128 => 16]) = 0
getsockname(4, {sa_family=AF_INET, sin_port=htons(53362), sin_addr=inet_addr("172.20.2.70")}, [128 => 16]) = 0

kubectl exec curl-pod -- strace -e trace=getsockopt curl -s webpod 
getsockopt(4, SOL_SOCKET, SO_ERROR, [0], [4]) = 0 # 소켓 연결 성공

# strace 로 IP 변환 확인이 어려운 이유 by ChatGPT
1. Socket LB는 커널 BPF 레벨에서 처리
- Cilium의 Socket LB (sock_ops / sk_msg / sockmap) 기능은 유저스페이스가 아닌 커널의 BPF 프로그램에서 소켓 수준에서 백엔드로 리디렉션합니다.
- strace는 시스템 콜 수준(ex: connect, sendto, recvmsg)만 추적하며, 커널 내부 동작 (BPF redirect) 은 보이지 않습니다.

2. strace에서 보이는 정보는 리디렉션 이후 상태일 뿐
- connect() 시 service IP로 요청했지만, 실제 커널에서는 이미 BPF 프로그램이 backend로 소켓을 연결했기 때문에,
- 유저 공간에서는 connect(backend-ip)처럼 보이지 않습니다.
```

(심화) strace : 시스템 콜 트레이싱 도구(디버거) , 시스템 콜 상태(커널 모드로 실행 중이거나 대기 상태)
```bash
# 중단점 트레이싱 : -ttt(첫 열에 기준시간으로부터 흐른 시간 표시) , -T(마지막 필드 time에 시스템 콜에 걸린 시간을 표시) , -p PID(프로세스 ID가 PID 인 프로세스를 트레이싱)
strace -ttt -T -p 1884

# 시스템 콜별 통계
strace -c -p 1884

# 그냥 사용해보기
strace ls

# 옵션 사용해보기 : -s(출력 string 결과 최댓값 지정), -tt(첫 열에 기준시간으로부터 흐른 시간 표시, ms단위), -f(멀티 스레드,멀티 프로레스의 자식 프로세스의 시스템 콜 추적)
strace -s 65535 -f -T -tt -o <파일명> -p <pid>

# hostname 명령 분석하기 : -o <파일명> 출력 결과를 파일로 떨구기
strace -s 65535 -f -T -tt -o hostname_f_trace hostname -f

# 특정 이벤트 : -e
strace -e trace=connect curl ipinfo.io
```

### ISSUE
(정보) 소켓 기반 로드밸런싱 사용 시 Istio(EnvoyProxy)와 같은 사이드카 우회 문제
![[Pasted image 20250713224927.png]]

- 해결 방안 : 파드 네임스페이스에서는 소켓 기반 로드밸런싱을 사용하지 않는다 → 호스트 네임스페이스만 사용하게 설정!
    - 단, HTTP 경우 Envoy 의 HTTP 필터가 HTTP 패킷의 host 헤더로 필터링하여 패킷의 목적지 주소가 서비스 IP에서 백엔드 IP로 변환이 잘된다
    - 하지만, HTTP가 아닌 TCP 서비스(예. Telnet 등)은 위 환경에서 문제가 발생한다.


```bash
# 설정
VERSION=1.11.2
helm upgrade cilium cilium/cilium --version $VERSION --namespace kube-system --reuse-values \\
  --set hostServices.hostNamespaceOnly=true
kubectl -n kube-system rollout restart ds/cilium

cilium config view | grep bpf-lb-sock-hostns-only
bpf-lb-sock-hostns-only                        true

# 확인
# 지속적으로 접속 트래픽 발생
while true; do kubectl exec netpod -- curl -s $SVCIP | grep Hostname;echo "-----";sleep 1;done

# 파드에서 SVC(ClusterIP) 접속 시 tcpdump 로 확인 >> 파드 내부 캡쳐인데, SVC(10.108.12.195) 트래픽이 보인다!
kubectl exec netpod -- tcpdump -enni any -q
	21:51:45.615174 eth0  Out ifindex 17 fe:9b:8a:a8:6e:83 172.16.0.245.41880 > 10.98.168.249.80: tcp 0
	21:51:45.615613 eth0  In  ifindex 17 7a:f4:08:6b:50:e5 10.98.168.249.80 > 172.16.0.245.41880: tcp 0
...
```


socket-LB Issue : Service ClusterIP로 NFS and SMB mounts(Longhorn, Portworx, and Robin) 사용하는 경우 마운트 X - [Docs](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#limitations) , [Issue](https://github.com/cilium/cilium/issues/21541)
- Cilium’s eBPF kube-proxy replacement relies upon the socket-LB feature which uses eBPF cgroup hooks to implement the service translation. Using it with libceph deployments currently requires support for the getpeername(2) hook address translation in eBPF, which is only available for kernels v5.8 and higher.
    
- NFS and SMB mounts may break when mounted to a `Service` cluster IP while using socket-LB. This issue is known to impact Longhorn, Portworx, and Robin, but may impact other storage systems that implement `ReadWriteMany` volumes using this pattern. To avoid this problem, ensure that the following commits are part of your underlying kernel:
    
    - `0bdf399342c5 ("net: Avoid address overwrite in kernel_connect")`
    - `86a7e0b69bd5 ("net: prevent rewrite of msg_name in sock_sendmsg()")`
    - `01b2885d9415 ("net: Save and restore msg_namelen in sock_sendmsg")`
    - `cedc019b9f26 ("smb: use kernel_connect() and kernel_bind()")` (SMB only)
    
    These patches have been backported to all stable kernels and some distro-specific kernels: _⇒ 아래 버전 이상 시 해결_
    
    - Ubuntu: `5.4.0-187-generic`, `5.15.0-113-generic`, `6.5.0-41-generic` or newer.
    - RHEL 8: `4.18.0-553.8.1.el8_10.x86_64` or newer (RHEL 8.10+).
    - RHEL 9: `kernel-5.14.0-427.31.1.el9_4` or newer (RHEL 9.4+).
    
    For a more detailed discussion see [GitHub issue 21541](https://github.com/cilium/cilium/issues/21541).

----------

# 도전과제
파드 내 Socket-Based LB 기법이 문제가 되는 환경을 재연(istio, Longhorn 등)해보고, 파드내 Socket LB를 Off해서 해결해보자

-----------

