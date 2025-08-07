# Encapsulation VXLAN 설정 - [Docs](https://docs.cilium.io/en/stable/network/concepts/routing/)
```bash 
# [커널 구성 옵션] Requirements for Tunneling and Routing
grep -E 'CONFIG_VXLAN=y|CONFIG_VXLAN=m|CONFIG_GENEVE=y|CONFIG_GENEVE=m|CONFIG_FIB_RULES=y' /boot/config-$(uname -r)
CONFIG_FIB_RULES=y # 커널에 내장됨
CONFIG_VXLAN=m # 모듈로 컴파일됨 → 커널에 로드해서 사용
CONFIG_GENEVE=m # 모듈로 컴파일됨 → 커널에 로드해서 사용

#  커널 로드
lsmod | grep -E 'vxlan|geneve'
modprobe vxlan # modprobe geneve
lsmod | grep -E 'vxlan|geneve'

for i in w1 w0 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo modprobe vxlan ; echo; done
for i in w1 w0 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo lsmod | grep -E 'vxlan|geneve' ; echo; done


# k8s-w1 노드에 배포된 webpod 파드 IP 지정
export WEBPOD1=$(kubectl get pod -l app=webpod --field-selector spec.nodeName=k8s-w1 -o jsonpath='{.items[0].status.podIP}')
echo $WEBPOD1

# 반복 ping 실행해두기
kubectl exec -it curl-pod -- ping $WEBPOD1


# 업그레이드
helm upgrade cilium cilium/cilium --namespace kube-system --version 1.18.0 --reuse-values \
  --set routingMode=tunnel --set tunnelProtocol=vxlan \
  --set autoDirectNodeRoutes=false --set installNoConntrackIptablesRules=false

kubectl rollout restart -n kube-system ds/cilium


# 설정 확인
cilium features status
cilium features status | grep datapath_network

kubectl exec -it -n kube-system ds/cilium -- cilium status | grep ^Routing
cilium config view | grep tunnel

# cilium_vxlan 확인
ip -c addr show dev cilium_vxlan
for i in w1 w0 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i ip -c addr show dev cilium_vxlan ; echo; done

# 라우팅 정보 확인 : k8s node 간 다른 네트워크 대역에 있더라도, 파드의 네트워크 대역 정보가 라우팅에 올라왔다!
ip -c route | grep cilium_host
172.20.0.238 dev cilium_host proto kernel scope link 
172.20.0.0/24 via 172.20.0.238 dev cilium_host proto kernel src 172.20.0.238 
172.20.1.0/24 via 172.20.0.238 dev cilium_host proto kernel src 172.20.0.238 mtu 1450 
172.20.2.0/24 via 172.20.0.238 dev cilium_host proto kernel src 172.20.0.238 mtu 1450 

ip route get 172.20.1.10
ip route get 172.20.2.10

for i in w1 w0 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i ip -c route | grep cilium_host ; echo; done


# cilium 파드 이름 지정
export CILIUMPOD0=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-ctr -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD1=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w1  -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD2=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w0  -o jsonpath='{.items[0].metadata.name}')
echo $CILIUMPOD0 $CILIUMPOD1 $CILIUMPOD2

# router 역할 IP 확인
kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- cilium status --all-addresses | grep router
kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- cilium status --all-addresses | grep router
kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- cilium status --all-addresses | grep router

#
kubectl exec -n kube-system ds/cilium -- cilium-dbg bpf ipcache list
kubectl exec -n kube-system $CILIUMPOD0 -- cilium-dbg bpf ipcache list
kubectl exec -n kube-system $CILIUMPOD1 -- cilium-dbg bpf ipcache list
kubectl exec -n kube-system $CILIUMPOD2 -- cilium-dbg bpf ipcache list

#
kubectl exec -n kube-system ds/cilium -- cilium-dbg bpf socknat list
kubectl exec -n kube-system $CILIUMPOD0 -- cilium-dbg bpf socknat list
kubectl exec -n kube-system $CILIUMPOD1 -- cilium-dbg bpf socknat list
kubectl exec -n kube-system $CILIUMPOD2 -- cilium-dbg bpf socknat list

```

# 파드 간 통신 확인 : gyullbb님이 통신 과정 (코드) 분석을 먼저 해주셨네요 - [Blog](https://velog.io/@_gyullbb/Cilium-Networking-1-IPAM-Routing-Masquerading)

![[Pasted image 20250803204614.png]]

```bash
# 통신 확인
kubectl exec -it curl-pod -- curl webpod | grep Hostname
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s --connect-timeout 1 webpod | grep Hostname; echo "---" ; sleep 1; done'

# k8s-w0 노드에 배포된 webpod 파드 IP 지정
export WEBPOD=$(kubectl get pod -l app=webpod --field-selector spec.nodeName=k8s-w0 -o jsonpath='{.items[0].status.podIP}')
echo $WEBPOD

# 신규 터미널 [router]
tcpdump -i any udp port 8472 -nn

# 
kubectl exec -it curl-pod -- ping -c 2 -w 1 -W 1 $WEBPOD

# 신규 터미널 [router] : 라우팅이 어떻게 되는가?
tcpdump -i any icmp -nn
08:49:29.969241 eth1  In  IP 192.168.10.100.54386 > 192.168.20.100.8472: OTV, flags [I] (0x08), overlay 0, instance 23926
IP 172.20.0.44 > 172.20.2.43: ICMP echo request, id 103, seq 1, length 64
08:49:29.969263 eth2  Out IP 192.168.10.100.54386 > 192.168.20.100.8472: OTV, flags [I] (0x08), overlay 0, instance 23926
IP 172.20.0.44 > 172.20.2.43: ICMP echo request, id 103, seq 1, length 64
08:49:29.970066 eth2  In  IP 192.168.20.100.36993 > 192.168.10.100.8472: OTV, flags [I] (0x08), overlay 0, instance 12438
IP 172.20.2.43 > 172.20.0.44: ICMP echo reply, id 103, seq 1, length 64
08:49:29.970068 eth1  Out IP 192.168.20.100.36993 > 192.168.10.100.8472: OTV, flags [I] (0x08), overlay 0, instance 12438
IP 172.20.2.43 > 172.20.0.44: ICMP echo reply, id 103, seq 1, length 64


# 반복 접속
kubectl exec -it curl-pod -- curl webpod | grep Hostname
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s --connect-timeout 1 webpod | grep Hostname; echo "---" ; sleep 1; done'

# 신규 터미널 [router]
tcpdump -i any udp port 8472 -w /tmp/vxlan.pcap
tshark -r /tmp/vxlan.pcap -d udp.port==8472,vxlan
termshark -r /tmp/vxlan.pcap
termshark -r /tmp/vxlan.pcap -d udp.port==8472,vxlan

# 신규 터미널 [k8s-ctr] hubble flow log 모니터링 : overlay 통신 모드 확인!
hubble observe -f --protocol tcp --pod curl-pod
Aug  3 00:02:26.403: default/curl-pod:40332 (ID:23926) -> default/webpod-697b545f57-76lt6:80 (ID:12438) to-overlay FORWARDED (TCP Flags: SYN)
Aug  3 00:02:26.405: default/curl-pod:40332 (ID:23926) <- default/webpod-697b545f57-76lt6:80 (ID:12438) to-endpoint FORWARDED (TCP Flags: SYN, ACK)
Aug  3 00:02:26.405: default/curl-pod:40332 (ID:23926) -> default/webpod-697b545f57-76lt6:80 (ID:12438) to-overlay FORWARDED (TCP Flags: ACK)

```


# Pod에서 빠져나갈 때,
![[Pasted image 20250803204655.png]]

# Pod로 인입될 때,
![[Pasted image 20250803204737.png]]

정리 : 노드 간 같은 네트워크 대역에 있어도, 모든 파드 들간 통신은 Overlay Network을 사용 (50바이트 헤더 추가, 처리를 위한 컴퓨팅 리소스)


# 도전과제2 - VXLAN 대신 GENEVE 모드로 변경 설정 후 설정 상태와 통신 트래픽을 확인해보세요.

^da9cfb



# 도전과제3 MTU 1450 의미는 무엇이고, 실제 환경에서는 고려해야 될 부분들은 어떤것이 있을까요? (참고) MTU, MSS - [Blog](https://aws.amazon.com/ko/blogs/tech/aws-mtu-mss-pmtud/)

^feb015

```bash
# -M do : Don't Fragment (DF) 플래그를 설정하여 조각화 방지
# -s 1472 : 페이로드(payload) 크기, 즉 ICMP 데이터 크기
kubectl exec -it curl-pod -- ping -M do -s 1500 $WEBPOD
PING 172.20.2.43 (172.20.2.43) 1500(1528) bytes of data.
ping: sendmsg: Message too large
```
