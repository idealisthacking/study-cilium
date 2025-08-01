Masquerading 소개 : (구현 방식) eBPF-based vs iptables-based - [Docs](https://docs.cilium.io/en/stable/network/concepts/masquerading/)

![[Pasted image 20250728002356.png]]

- 포드에 사용되는 IPv4 주소는 일반적으로 RFC1918 개인 주소 블록에서 할당되므로 공개적으로 라우팅할 수 없습니다.
- Cilium은 **노드의 IP 주소**가 이미 네트워크에서 라우팅 가능하기 때문에 **클러스터를 떠나는 모든 트래픽**의 **소스 IP 주소**를 자동으로 **masquerade** 합니다.
- 만약 masquerade 기능을 OFF 하려면, `enable-ipv4-masquerade: false` , `enable-ipv6-masquerade: false`

- 기본 동작은 로컬 노드의 IP 할당 CIDR 내에서 모든 목적지를 제외하는 것입니다.
- 더 넓은 네트워크에서 포드 IP를 라우팅할 수 있는 경우, 해당 네트워크는 `ipv4-native-routing-cidr: 10.0.0/8` (또는 IPv6 주소의 경우 `ipv6-native-routing-cidr: fd00:/100`) 옵션을 사용하여 지정할 수 있습니다. 이 경우 해당 CIDR 내의 모든 목적지는 masquerade 되지 않습니다.

- 구현 방식 : eBPF-based vs iptables-based
- **eBPF-based :** `bpf.masquerade=true`
    - By default, **BPF masquerading** also enables the **BPF Host-Routing** mode. See [eBPF Host-Routing](https://docs.cilium.io/en/stable/operations/performance/tuning/#ebpf-host-routing) for benefits and limitations of this mode.
    - Masquerading은 eBPF Masquerading 프로그램을 실행하는 장치에서만 수행할 수 있습니다.
    - 이는 출력 장치가 프로그램을 실행하는 경우 포드에서 외부 주소로 전송된 패킷이 Masquerading(출력 장치 IPv4 주소로)된다는 것을 의미합니다.
```bash
#
kubectl exec -it -n kube-system ds/cilium -c cilium-agent  -- cilium status | grep Masquerading
Masquerading:            BPF   [eth0, eth1]   172.20.0.0/16 [IPv4: Enabled, IPv6: Disabled]
```

- 지정되지 않은 경우, 프로그램은 BPF NodePort 장치 감지 메커니즘에 의해 선택된 장치에 자동으로 연결됩니다.
- 이를 수동으로 변경하려면 `devices` helm 옵션을 사용하세요.
- The eBPF-based masquerading can masquerade packets of the following L4 protocols: **TCP, UDP, ICMP**
- 기본적으로 `ipv4-native-routing-cidr` 범위를 벗어난 IP 주소로 향하는 포드의 모든 패킷은 **Masquerading**되지만, 다른 (클러스터) 노드(**Node IP**)로 향하는 패킷은 제외됩니다. eBPF 마스커딩이 활성화되면 포드에서 클러스터 노드의 **External IP**로의 트래픽도 마스커딩되지 않습니다.

```bash
#
cilium config view  | grep ipv4-native-routing-cidr
ipv4-native-routing-cidr                          172.20.0.0/16

# 노드 IP로 통신 시 확인
tcpdump -i eth1 icmp -nn
kubectl exec -it curl-pod -- ping 192.168.10.101
...
```

좀 더 정교한 설정은 (Cilium 의 eBPF 구현)

ip-masq-agent

를 통해서 가능합니다. 아래 실습에서 자세히 다루보자 - [Docs](https://github.com/kubernetes-sigs/ip-masq-agent)

- **iptables-based**
    - 이것은 모든 커널 버전에서 작동할 **레거시** 구현입니다. This is the **legacy implementation** that will work on all kernel versions.
    - The default behavior will masquerade all traffic leaving on a non-Cilium network device. This typically leads to the correct behavior.
    - In order to limit the network interface on which masquerading should be performed, the option `egress-masquerade-interfaces: eth0` can be used.
    - For the advanced case where the routing layer would select different source addresses depending on the destination CIDR, the option `enable-masquerade-to-route-source: "true"` can be used in order to masquerade to the source addresses rather than to the primary interface address.

[[3Week/실습 환경|실습 환경]] 참조 


## 현재 상태
```bash
# 현재 설정 확인
kubectl exec -it -n kube-system ds/cilium -c cilium-agent  -- cilium status | grep Masquerading
Masquerading:            BPF   [eth0, eth1]   172.20.0.0/16 [IPv4: Enabled, IPv6: Disabled]
#
cilium config view  | grep ipv4-native-routing-cidr
ipv4-native-routing-cidr                          172.20.0.0/16

# 통신 확인
kubectl exec -it curl-pod -- curl -s webpod | grep Hostname
```

## router eth1 192.168.10.200 통신 확인
```bash
# 터미널 2개 사용
[k8s-ctr] tcpdump -i eth1 icmp -nn 혹은 hubble observe -f --pod curl-pod
[router] tcpdump -i eth1 icmp -nn

# router eth1 192.168.10.200 로 ping >> IP 확인해보자!
kubectl exec -it curl-pod -- ping 192.168.10.101
kubectl exec -it curl-pod -- ping 192.168.10.200
...

---
# 터미널 2개 사용
[k8s-ctr] tcpdump -i eth1 tcp port 80 -nnq 혹은 hubble observe -f --pod curl-pod
[router] tcpdump -i eth1 tcp port 80 -nnq

# router eth1 192.168.10.200 로 curl >> IP 확인해보자!
kubectl exec -it curl-pod -- curl -s webpod
kubectl exec -it curl-pod -- curl -s webpod
kubectl exec -it curl-pod -- curl -s 192.168.10.200
...
```

## router loop1/2 통신 확인
```bash
# router
ip -br -c -4 addr
loop1            UNKNOWN        10.10.1.200/24
loop2            UNKNOWN        10.10.2.200/24
...

# k8s-ctr
ip -c route | grep static
10.10.0.0/16 via 192.168.10.200 dev eth1 proto static


# 터미널 2개 사용
[k8s-ctr] tcpdump -i eth1 tcp port 80 -nnq 혹은 hubble observe -f --pod curl-pod
[router] tcpdump -i eth1 tcp port 80 -nnq


# router eth1 192.168.10.200 로 curl >> IP 확인해보자!
kubectl exec -it curl-pod -- curl -s 10.10.1.200
kubectl exec -it curl-pod -- curl -s 10.10.2.200

```

## (Cilium 의 eBPF 구현) ip-masq-agent 설정 - [Docs](https://github.com/kubernetes-sigs/ip-masq-agent)

- The eBPF-based ip-masq-agent supports the `nonMasqueradeCIDRs`, `masqLinkLocal`, and `masqLinkLocalIPv6` options set in a configuration file.
- A packet sent from a pod to a destination which belongs to any CIDR from the `nonMasqueradeCIDRs` is not going to be masqueraded. _→ 사내망과 NAT 없는 통신 필요 시 해당 설정에 대역들 추가 할 것! , ClusterMesh 시 Native-Routing 설정 시에도 활용._
- If the configuration file is empty, the agent will provision the following non-masquerade CIDRs: _→ 설정값 없을 경우 기본 값!_

```
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
100.64.0.0/10
192.0.0.0/24
192.0.2.0/24
192.88.99.0/24
198.18.0.0/15
198.51.100.0/24
203.0.113.0/24
240.0.0.0/4
```

In addition, if the masqLinkLocal is not set or set to false, then 169.254.0.0/16 is appended to the non-masquerade CIDRs list.

## ipMasqAgent 설정
```bash
# 아래 설정값은 cilium 데몬셋 자동 재시작됨
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
--set ipMasqAgent.enabled=true --set ipMasqAgent.config.nonMasqueradeCIDRs='{10.10.1.0/24,10.10.2.0/24}'

cilium hubble port-forward&

# ip-masq-agent configmap 생성 확인
kubectl get cm -n kube-system ip-masq-agent -o yaml | yq
kc describe cm -n kube-system ip-masq-agent 
k9s 

#
cilium config view  | grep -i ip-masq
enable-ip-masq-agent                              true

#
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg bpf ipmasq list
IP PREFIX/ADDRESS   
10.10.1.0/24             
10.10.2.0/24             
169.254.0.0/16
```