# Cilium Endpoint Slices (CES) beta - [Youtube](https://www.youtube.com/watch?v=yKPNmhckJHY)
## K8S Cilium 기본 환경 : 5000개 노드 x 100개 엔드포인트 = 500,000 Watch Update
![[Pasted image 20250828112040.png]]
- Load on Kube-API Server / Etcd by each Cilium-agentac
- Pod Network Provision SLA
- Per-Node CPU/Memory Impact by Cilium-Agent daemonsets

## Cilium Endpoint Slices (CES) beta! - [Docs](https://docs.cilium.io/en/stable/network/kubernetes/ciliumendpointslice/)
![[Pasted image 20250828112103.png]]

## 사전 조건
- Make sure that CEPs are enabled (the `-disable-endpoint-crd` flag is not set to `true`)
- Make sure you are not relying on the Egress Gateway which is not compatible with CES (see Egress Gateway [Incompatibility with other features](https://docs.cilium.io/en/stable/network/egress-gateway/egress-gateway/#egress-gateway-incompatible-features))

## 설정
```bash
# CiliumEndpoint (CEP) per pod : All agents watch for all CEPs, Maps IP to Identity for policy.
kubectl get ciliumendpoints.cilium.io -A
kubectl get ciliumendpoints.cilium.io -A | wc -l
kubectl get crd

#
helm upgrade cilium cilium/cilium --version 1.18.1 --namespace kube-system --reuse-values \
--set ciliumEndpointSlice.enabled=true

kubectl rollout restart -n kube-system deployment cilium-operator
kubectl rollout restart -n kube-system ds/cilium 

# Bactches CEPs into CiliumEndpointSlice CRDs : Significantly fewer watch events.
kubectl get crd
kubectl get ciliumendpointslices.cilium.io -A | wc -l
kubectl get ciliumendpointslices.cilium.io -A
NAME                  AGE
ces-g7lwptqjy-m78ty   40s
ces-mx8x5cv89-7nzrg   40s
ces-r2l9hwd46-4hq92   40s
```

## CES 동작 : cilium-operator 에 ces-controller
![[Pasted image 20250828112148.png]]

## 적용 후 효과 : 1000개 노드 등 - APIServer latency 87% 감소
![[Pasted image 20250828112204.png]]

# eBPF Host Routing , Netkit - [Youtube](https://www.youtube.com/watch?v=yKPNmhckJHY) , [Youtube](https://www.youtube.com/watch?v=cKPW67D7X10)2
## Basic container netwokring
![[Pasted image 20250828112227.png]]

## eBPF Host Routing : bpf_redirect_peer 함수를 통해 호스트 NET NS 도착 시 → 바로 파드의 NET NS 로 전달.
![[Pasted image 20250828112244.png]]
![[Pasted image 20250828112256.png]]
![[Pasted image 20250828112316.png]]

## 하지만 Egress 는 호스트 네트워크 스택(per-CPU backlog queue)을 통과해야한다..
![[Pasted image 20250828112352.png]]
![[Pasted image 20250828112409.png]]

```bash
#
kubectl exec -it -n kube-system ds/cilium -- cilium status | grep Routing
KubeProxyReplacement:    True   [eth0    172.18.0.2 fc00:f853:ccd:e793::2 fe80::42:acff:fe12:2 (Direct Routing)]
Routing:                 Network: Native   Host: BPF
```

## Netkit devices : veth 와 유사(쌍), ingress + egress 큐(CPU) 대기 없이 모두 빠르게 전달 가능.
![[Pasted image 20250828112443.png]]
![[Pasted image 20250828112456.png]]
![[Pasted image 20250828112508.png]]
![[Pasted image 20250828112520.png]]
## netkit 설정 시도 : 커널 6.8 이상 , eBPF host-routing
```bash
#
docker exec -it myk8s-control-plane uname -r
6.10.14-linuxkit

#
helm upgrade cilium cilium/cilium --version 1.18.1 --namespace kube-system --reuse-values \
--set bpf.datapathMode=netkit

kubectl rollout restart -n kube-system deployment cilium-operator
kubectl rollout restart -n kube-system ds/cilium 

#
kubectl logs -n kube-system cilium-k84hl
time=2025-08-24T09:33:36.486143842Z level=fatal source=/go/src/github.com/cilium/cilium/pkg/logging/slog.go:159 
.. msg="netkit devices need kernel 6.7.0 or newer and CONFIG_NETKIT"

#
helm upgrade cilium cilium/cilium --version 1.18.1 --namespace kube-system --reuse-values \
--set bpf.datapathMode=veth

kubectl exec -it -n kube-system ds/cilium -- cilium status | grep 'Device Mode'
Device Mode:             veth

```


# 2023 eBPF XDP-based Acceleration Service Load-Balancer - [Youtube](https://www.youtube.com/watch?v=AVEBcZc0YsQ)
## XDP & TC Layer 에서 LB 처리 시 : L4 LB XDP vs IPVS 비교
![[Pasted image 20250828112608.png]]

## XDP 설정 확인 : HW 지원 필요.
```bash
#
docker exec -it myk8s-control-plane apt install bpftool -y
docker exec -it myk8s-control-plane bpftool net
docker exec -it myk8s-control-plane bpftool link

# 설정 시 아래 항목에 활성화
kubectl exec -it -n kube-system ds/cilium -- cilium status --verbose | grep -i xdp
  XDP Acceleration:     Disabled
```

# 2023 BIG TCP (IPv6 IPv4) - [Youtube](https://www.youtube.com/watch?v=AVEBcZc0YsQ) , [Isovalent](https://www.youtube.com/watch?v=oVAu8M6RZ3k) , [Blog](https://isovalent.com/blog/post/big-tcp-on-cilium/) , [OnlineLab](https://isovalent.com/labs/cilium-big-tcp/)
![[Pasted image 20250828112651.png]]

- **배경** : 100Gbps 이상 환경
    - Cilium을 도입한 많은 조직(클라우드 공급업체, 금융 기관, 통신 공급업체)은 모두 공통점이 있습니다. 이들은 모두 네트워크에서 최대한 많은 성능을 끌어내고자 하며 끊임없이 미미한 성능 향상을 모색합니다.
    - 이러한 조직들은 100Gbps 이상의 네트워크를 구축하고 있지만, 100Gbps 네트워크 어댑터를 도입하면서 피할 수 없는 과제가 발생합니다. 바로 **CPU**가 **초당 800만 개의 패킷**(MTU가 **1,538**바이트라고 가정할 때)을 어떻게 처리할 수 있느냐는 것입니다. 따라서 시스템이 패킷당 처리할 수 있는 시간은 **120나노초**밖에 남지 않아 비현실적입니다. 또한 인터페이스에서 상위 프로토콜 계층까지 이 모든 패킷을 처리하는 데 상당한 오버헤드가 발생합니다.
    - 분명히 이 제한은 새로운 것이 아니며 **패킷을 일괄 처**리하여 해결되었습니다. Linux 스택에서는 [GRO 를 통해 해결됩니다.](https://lwn.net/Articles/358910/)(일반 수신 오프로드) 및 [TSO](https://www.kernel.org/doc/Documentation/networking/segmentation-offloads.txt)(Transmit Segmentation Offload, 전송 분할 오프로드). **수신** 측에서는 **GRO**가 패킷을 스택 내에서 **64KB 크**기의 **초대형 패킷으로 그룹화**하여 네트워킹 스택으로 전달합니다. 마찬가지로, **송신** 측에서는 **TSO**가 NIC가 처리할 수 있도록 **TCP 초대형 패킷을** 분할합니다.

![[Pasted image 20250828112735.png]]

- 초대형 64K 패킷이 도움이 되기는 하지만, 최신 CPU는 실제로 훨씬 더 큰 패킷을 처리할 수 있습니다.
- 하지만 **64K는 여전히 엄격한 제한**으로 남아 있었습니다. IP 패킷의 길이는 **16비트 필드에 옥텟 단위**로 지정됩니다. 따라서 **최대값은 65,535바이트(**64KB)입니다.
![[Pasted image 20250828112814.png]]

* 더 큰 패킷 크기를 지정할 수 있는 더 큰 헤더를 찾을 수 있다면 어떨까요? 패킷 크기가 커지면 오버헤드가 줄어들고 이론적으로 처리량이 향상되며 지연 시간도 단축됩니다.

# IPv6 를 통한 BIG TCP
- 이것이 바로 BIG TCP가 설계된 목적입니다. Google의 Linux 커널 전문가 Eric Dumazet과 Coco Li가 개발한 BIG TCP는 Linux 5.19 커널에 도입되었습니다.
- _특이하게도 BIG TCP 지원은 IPv4 보다_ 먼저 IPv6에 도입되었습니다 . 22년 된 RFC( [RFC2675)](https://datatracker.ietf.org/doc/html/rfc2675) 를 통해 IPv6에 도입하는 것이 더 쉬웠습니다.) IPv6 점보그램(64KB보다 큰 패킷)을 설명합니다.
- **IPv6**는 패킷에 삽입할 수 있는 **홉 바이 홉 헤더를 지원**합니다. 홉 바이 홉 확장 헤더에 페이로드 길이를 지정하고 (무시하려면 IPv6 헤더의 페이로드 길이 필드를 0으로 설정) 64K 제한을 해결할 수 있습니다.

![[Pasted image 20250828112934.png]]
Hop-by-Hop 확장 헤더는 페이로드 길이에 32비트 필드를 사용하는데, 이를 통해 최대 4GB 패킷을 사용할 수 있지만, 처음에는 제한이 64KB에서 512KB로 증가합니다.

![[Pasted image 20250828112957.png]]

- 교통 비유를 다시 말씀드리겠습니다. 단일 차량(표준 TCP/IP)에서 **버스(GSO/GRO)**로, 그리고 고**속 열차(BIG TCP)**로 옮겨갔습니다.
- IPv6를 통한 BIG TCP 지원은 Cilium 1.13에서 도입되었습니다.


# IPv4 BIG TCP
- IPv4에 대한 BIG TCP 지원은 Linux 커널 6.3에서 도입되었습니다. 자세한 내용은 Linux 커널 [메일링 리스트 에서 확인할 수 있습니다.](https://lore.kernel.org/netdev/cover.1674921359.git.lucien.xin@gmail.com/)
- 하지만 간단히 말해서 IPv4에는 Hop-by-hop 확장 헤더가 없으므로, `skb->len`Linux 개발자가 참조하는 **소켓 버퍼에 저장된 데이터 페이로드의 길이**를 사용하여 더 큰 패킷 크기를 지정합니다.
- [IPv4를 통한 BIG TCP는 Cilium 1.14](https://isovalent.com/blog/post/cilium-release-114/#h-big-tcp-for-ipv4-beta) 에 도입되었습니다.

# 점보 프레임 vs BIG TCP
- 이 블로그 게시물을 700단어나 읽었는데요, 왜 아직도 MTU(최대 전송 단위)에 대해 언급하지 않았는지 궁금하실 겁니다. 또한 다음과 같은 질문도 하실 수 있습니다. 1) BIG TCP와 점보 프레임은 같은 것인가요? 2) BIG TCP를 지원하기 위해 모든 네트워크 장치의 MTU를 변경해야 할까요?
- **점보 프레임은 이더넷 프레임 크기를 6배 증가**시켜(표준 **1,500**바이트에서 무려 **9,000**바이트로) **BIG TCP와 마찬가지로 성능을 크게 향상**시킵니다. OSI 계층의 2계층에 위치하므로 IPv4와 IPv6 모두에 적용됩니다. 하지만 가장 큰 차이점은 **점보 프레임은 물리적 네트워크**를 통과하며, **이더넷 장비가 점보 프레임을 지원**해야 하고, **프레임이 통과하는 모든 네트워크 인터페이스도 이에 맞게 구성**되어야 한다는 것입니다.
- **BIG TCP**는 **네트워크 장치의 MTU를 수정할 필요가 없습**니다. 이는 **Linux 네트워킹 스택 _내에서_ 작동하**도록 설계 되었으며, **GRO가 수신 시 패킷을 64KB의 초대형 패킷 대신 512KB의 초대형 패킷으로 집**계하고**, GSO가 송신 시 초대형 패킷을 생성하여 스택 아래로 푸**시하는 데 사용됩니다.
- 아직도 MTU 불일치 문제를 해결해야 한다는 악몽에 시달리는 전직 네트워크 엔지니어로서, 처음 이해했던 것과 달리 BIG TCP는 물리적 네트워크를 변경할 필요가 없다는 사실을 알게 되어 위안이 되었습니다.
- (참고)
    - MTU, MSS - [Blog](https://aws.amazon.com/ko/blogs/tech/aws-mtu-mss-pmtud/)
    - 류영무님이 실습 환경에서 MTU 9000 변경하여 적용 확인 및 Cilium MTU 관련 상세히 정리 - [Blog](https://ryusstory.tistory.com/entry/cilium-tunnelencapsulation%EB%AA%A8%EB%93%9C%EC%97%90%EC%84%9C-MTU%EC%A0%90%EB%B3%B4%ED%94%84%EB%A0%88%EC%9E%84)

# IPv6 BIG TCP 실습 따라해보자 : 지원되는 NIC(mlx4, mlx5) - [Docs](https://isovalent.com/blog/post/big-tcp-on-cilium/#big-tcp-on-cilium)
![[Pasted image 20250830000748.png]]

![[Pasted image 20250830000800.png]]
## 설정 시도 : NIC 지원 필요.
```bash
# TSO 확인 : 작동 위치 - 송신(TX), NIC
## TCP 레이어에서 큰 세그먼트를 보내면 NIC가 직접 분할
## CPU는 분할 부담을 덜고, NIC가 MTU에 맞게 처리
## NIC가 TSO를 지원해야 작동
ethtool -k eth1 | grep tcp-segmentation-offload
tcp-segmentation-offload: on

# GRO 확인 : 작동 위치 - 수신(RX), 커널
## 수신 시점에 여러 패킷을 하나의 큰 버퍼로 병합
## 커널에서 처리할 패킷 수를 줄여 CPU 부담 감소
## NIC 드라이버가 GRO를 지원해야 작동
ethtool -k eth1 | grep generic-receive-offload
generic-receive-offload: on

# GSO 확인 : 작동 위치 - 송신(TX), 커널 => 요거 대신, TSO 쓰는 듯
## 커널에서 큰 데이터 버퍼를 만들어 드라이버로 전달
## 드라이버나 NIC가 나중에 실제 MTU 단위로 잘라냄
## TSO가 없을 때 fallback 용도로 쓰임
ethtool -k eth1 | grep generic-segmentation-offload
generic-segmentation-offload: on

---
ip -details -c link show dev eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:e3:b1:8c brd ff:ff:ff:ff:ff:ff promiscuity 0  allmulti 0 minmtu 46 maxmtu 16110 addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 tso_max_size 65536 tso_max_segs 65535 gro_max_size 65536 parentbus pci parentdev 0000:00:09.0 
    altname enp0s9

ip -details -c link show dev lxce1621569242f
27: lxce1621569242f@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 42:39:b8:e8:e6:29 brd ff:ff:ff:ff:ff:ff link-netns cni-0f05c6f5-9bb2-8915-90b3-456648686924 promiscuity 0  allmulti 0 minmtu 68 maxmtu 65535 
    veth addrgenmode eui64 numtxqueues 4 numrxqueues 4 gso_max_size 65536 gso_max_segs 65535 tso_max_size 524280 tso_max_segs 65535 gro_max_size 65536 


ip -details -c addr show dev eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:e3:b1:8c brd ff:ff:ff:ff:ff:ff promiscuity 0  allmulti 0 minmtu 46 maxmtu 16110 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 tso_max_size 65536 tso_max_segs 65535 gro_max_size 65536 parentbus pci parentdev 0000:00:09.0 
    altname enp0s9
    inet 192.168.10.100/24 brd 192.168.10.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fee3:b18c/64 scope link 
       valid_lft forever preferred_lft forever

ip -details -c addr show dev lxce1621569242f
27: lxce1621569242f@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 42:39:b8:e8:e6:29 brd ff:ff:ff:ff:ff:ff link-netns cni-0f05c6f5-9bb2-8915-90b3-456648686924 promiscuity 0  allmulti 0 minmtu 68 maxmtu 65535 
    veth numtxqueues 4 numrxqueues 4 gso_max_size 65536 gso_max_segs 65535 tso_max_size 524280 tso_max_segs 65535 gro_max_size 65536 
    inet6 fe80::4039:b8ff:fee8:e629/64 scope link 
       valid_lft forever preferred_lft forever

---

#
helm upgrade cilium cilium/cilium --version 1.18.0 --namespace kube-system --reuse-values \
  --set enableIPv4BIGTCP=true

kubectl -n kube-system rollout restart ds/cilium

# IPv4 BIG TCP 옵션을 전환한 후 변경 사항을 적용하려면 Kubernetes Pods를 재시작해야 합니다!
# 이미 배포된 파드들 모두 재시작!

#
kubectl exec -it -n kube-system ds/cilium -- cilium status | grep -i big
IPv4 BIG TCP:            Enabled   [65536]
IPv6 BIG TCP:            Disabled

---
#
ip -details -c link show dev eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:e3:b1:8c brd ff:ff:ff:ff:ff:ff promiscuity 0  allmulti 0 minmtu 46 maxmtu 16110 addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 tso_max_size 65536 tso_max_segs 65535 gro_max_size 65536 parentbus pci parentdev 0000:00:09.0 
    altname enp0s9


ip -details -c link show dev lxce1621569242f
27: lxce1621569242f@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 42:39:b8:e8:e6:29 brd ff:ff:ff:ff:ff:ff link-netns cni-0f05c6f5-9bb2-8915-90b3-456648686924 promiscuity 0  allmulti 0 minmtu 68 maxmtu 65535 
    veth addrgenmode eui64 numtxqueues 4 numrxqueues 4 gso_max_size 65536 gso_max_segs 65535 tso_max_size 524280 tso_max_segs 65535 gro_max_size 65536 


ip -details -c addr show dev eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:e3:b1:8c brd ff:ff:ff:ff:ff:ff promiscuity 0  allmulti 0 minmtu 46 maxmtu 16110 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 tso_max_size 65536 tso_max_segs 65535 gro_max_size 65536 parentbus pci parentdev 0000:00:09.0 
    altname enp0s9
    inet 192.168.10.100/24 brd 192.168.10.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fee3:b18c/64 scope link 
       valid_lft forever preferred_lft forever

ip -details -c addr show dev lxce1621569242f
27: lxce1621569242f@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 42:39:b8:e8:e6:29 brd ff:ff:ff:ff:ff:ff link-netns cni-0f05c6f5-9bb2-8915-90b3-456648686924 promiscuity 0  allmulti 0 minmtu 68 maxmtu 65535 
    veth numtxqueues 4 numrxqueues 4 gso_max_size 65536 gso_max_segs 65535 tso_max_size 524280 tso_max_segs 65535 gro_max_size 65536 
    inet6 fe80::4039:b8ff:fee8:e629/64 scope link 
       valid_lft forever preferred_lft forever

#
docker exec -it myk8s-control-plane ip -d -j link show dev eth0    
[{"ifindex":35,"link_index":36,"ifname":"eth0","flags":["BROADCAST","MULTICAST","UP","LOWER_UP"],"mtu":65535,"qdisc":"noqueue","operstate":"UP","linkmode":"DEFAULT","group":"default","link_type":"ether","address":"02:42:ac:12:00:02","broadcast":"ff:ff:ff:ff:ff:ff","link_netnsid":0,"promiscuity":0,"allmulti":0,"min_mtu":68,"max_mtu":65535,"linkinfo":{"info_kind":"veth"},"inet6_addr_gen_mode":"eui64","num_tx_queues":8,"num_rx_queues":8,"gso_max_size":65536,"gso_max_segs":65535,"tso_max_size":524280,"tso_max_segs":65535,"gro_max_size":65536}]

#
docker exec -it myk8s-control-plane ip -d -j link show dev eth0 | jq -c '.[0].gso_max_size'
65536

#
ethtool -k eth1 
```

- Bandwidth Manager (optional, for BBR congestion control) : 정리 추가 예정
- eBPF clock probe to use jiffies for CT map : 정리 추가 예정

# `도전과제7` AWS EC2 상위 Spec EC2 등 NIC 에서 지원 가능한 환경에서 Cilium Tuning 설정 직접 적용해보고 성능 비교해보기

^895668

# `도전과제8` Cilium 공식문서에 Scalability report 의 테스트(200 노드) 따라해보기 - [Docs](https://docs.cilium.io/en/stable/operations/performance/scalability/report/)

^168b1a

