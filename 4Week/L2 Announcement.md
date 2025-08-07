
>[!note] Metal LB와 비슷함

# (참고) MetalLB Layer2 모드 - [Docs](https://metallb.universe.tf/concepts/layer2/)
![[Pasted image 20250803211241.png]]
![[Pasted image 20250803211253.png]]
![[Pasted image 20250803211313.png]]

![[Pasted image 20250803211329.png]]


# Cilium Layer 2 (L2) Announcement using ARP : - [Docs](https://docs.cilium.io/en/stable/network/l2-announcements/)

- Announcements은 로컬 영역 네트워크에서 서비스를 가시화하고 도달할 수 있도록 하는 기능입니다. 이 기능은 주로 사무실이나 캠퍼스 네트워크와 같은 BGP 기반 라우팅 없이 네트워크 내에서 온프레미스 배포를 목적으로 합니다.
- 이 기능을 사용하면 ExternalIPs 또는 LoadBalancer IP에 대한 ARP 쿼리에 응답합니다. 이러한 IP는 여러 노드에 있는 가상 IP(네트워크 장치에 설치되지 않음)이므로 각 서비스에 대해 한 번에 하나의 노드가 ARP 쿼리에 응답하고 MAC 주소로 응답합니다. 이 노드는 서비스 로드 밸런싱 기능으로 로드 밸런싱을 수행하여 north/south 로드 밸런서 역할을 합니다.
- 이 기능의 장점은 각 서비스가 고유한 IP를 사용할 수 있어 여러 서비스가 동일한 포트 번호를 사용할 수 있다는 점입니다. NodePort를 사용할 때 트래픽을 보낼 호스트를 결정하는 것은 클라이언트의 몫이며, 노드가 다운되면 IP+Port 콤보를 사용할 수 없게 됩니다. L2 Announcements를 통해 서비스 VIP는 다른 노드로 마이그레이션하기만 하면 계속 작동하게 됩니다.

![[Pasted image 20250803211606.png]]

![[Pasted image 20250803211625.png]]

- 제약사항
    - The feature currently **does not support IPv6/NDP.**
    - Due to the way L3 → L2 translation protocols work, one node receives all ARP requests for a specific IP, so no load balancing can happen before traffic hits the cluster.
    - The feature currently has **no traffic balancing mechanism** so nodes within the same policy might be asymmetrically **loaded**. For details see [Leader Election](https://docs.cilium.io/en/stable/network/l2-announcements/#l2-announcements-leader-election).
    - The feature is incompatible with the **`externalTrafficPolicy: Local`** on services as it may cause service IPs to be announced on nodes without pods causing traffic drops.

# [k8s 클러스터 외부] webpod 서비스를 LoadBalancer ExternalIP로 호출 with L2 Announcements

```bash
# 모니터링 : router VM
arping -i eth1 $LBIP -c 100000
Timeout
Timeout
60 bytes from 08:00:27:01:19:f3 (192.168.10.211): index=0 time=769.167 usec
60 bytes from 08:00:27:01:19:f3 (192.168.10.211): index=1 time=417.917 usec
...

# 설정 업그레이드
helm upgrade cilium cilium/cilium --namespace kube-system --version 1.18.0 --reuse-values \
   --set l2announcements.enabled=true && watch -d kubectl get pod -A

kubectl rollout restart -n kube-system ds/cilium

# 확인
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg config --all | grep EnableL2Announcements
EnableL2Announcements             : true

cilium config view | grep enable-l2
enable-l2-announcements                           true
enable-l2-neigh-discovery                         true


# 정책 설정 : arp 광고하게 될 service 와 node 지정(controlplane 제외) -> 설정 직후 arping 확인!
## 제약사항 : L2 ARP 모드에서 LB IPPool 은 같은 네트워크 대역에서만 유효. -> k8s-w0 을 제외한 이유. 포함 시 리더 노드 선정 시 동작 실패 상황 발생!
cat << EOF | kubectl apply -f -
apiVersion: "cilium.io/v2alpha1"  # not v2
kind: CiliumL2AnnouncementPolicy
metadata:
  name: policy1
spec:
  serviceSelector:
    matchLabels:
      app: webpod
  nodeSelector:
    matchExpressions:
      - key: kubernetes.io/hostname
        operator: NotIn
        values:
          - k8s-w0
  interfaces:
  - ^eth[1-9]+
  externalIPs: true
  loadBalancerIPs: true
EOF

```

```bash
# 확인
kubectl -n kube-system get lease
kubectl -n kube-system get lease | grep "cilium-l2announce"
NAME                                   HOLDER                                                                      AGE
cilium-l2announce-default-webpod      k8s-w1                                                                      62s
...

# 현재 리더 역할 노드 확인
kubectl -n kube-system get lease/cilium-l2announce-default-webpod -o yaml | yq
...
spec:
  acquireTime: "2025-06-18T09:46:41.081674Z"
  holderIdentity: k8s-w1
  leaseDurationSeconds: 15
  leaseTransitions: 0
  renewTime: "2025-06-18T09:48:45.753123Z"


# cilium 파드 이름 지정
export CILIUMPOD0=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-ctr -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD1=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w1  -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD2=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w0  -o jsonpath='{.items[0].metadata.name}')
echo $CILIUMPOD0 $CILIUMPOD1 $CILIUMPOD2

# 현재 해당 IP에 대한 리더가 위치한 노드의 cilium-agent 파드 내에서 정보 확인
kubectl exec -n kube-system $CILIUMPOD0 -- cilium-dbg shell -- db/show l2-announce
kubectl exec -n kube-system $CILIUMPOD1 -- cilium-dbg shell -- db/show l2-announce
IP               NetworkInterface
192.168.10.211   eth1
kubectl exec -n kube-system $CILIUMPOD2 -- cilium-dbg shell -- db/show l2-announce

# 로그 확인
kubectl -n kube-system logs ds/cilium | grep "l2"


# router VM : LBIP로 curl 요청 확인
arping -i eth1 $LBIP -c 1000
curl --connect-timeout 1 $LBIP

# VIP 에 대한 mac 주소가 리더 노드의 mac 주소와 동일함을 확인
arp -a
? (192.168.10.211) at 08:00:27:01:19:f3 [ether] on eth1
? (192.168.10.101) at 08:00:27:01:19:f3 [ether] on eth1
? (192.168.10.100) at 08:00:27:64:bf:57 [ether] on eth1
...

curl -s $LBIP
curl -s $LBIP | grep Hostname
curl -s $LBIP | grep RemoteAddr

# 리더 노드가 아닌 다른 노드에 webpod 통신 시, SNAT 됨 : arp 동작(리더 노드)으로 인한 제약 사항
while true; do curl -s --connect-timeout 1 $LBIP | grep RemoteAddr; sleep 0.1; done

```

# L2 Announcement 리더 노드에 장애 주입 후 Failover 확인

```bash
# 신규 터미널 (router) : 반복 호출
while true; do curl -s --connect-timeout 1 $LBIP | grep RemoteAddr; sleep 0.1; done


# 현재 리더 노드 확인
kubectl -n kube-system get lease | grep "cilium-l2announce"
NAME                                   HOLDER                                                                      AGE
cilium-l2announce-default-webpod      k8s-w1                                                                      62s
...

# 리더 노드 강제 reboot
sshpass -p 'vagrant' ssh vagrant@k8s-w1  sudo reboot
혹은
sshpass -p 'vagrant' ssh vagrant@k8s-ctr sudo reboot


# 신규 터미널 (router) : arp 변경(갱신) 확인
arp -a

```

# 도전과제4 L2 Pod Announcements 설정 및 동작 확인해보자 - [Docs](https://docs.cilium.io/en/stable/network/l2-announcements/#l2-pod-announcements)

^59ea22

