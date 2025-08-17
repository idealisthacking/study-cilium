Cilium BGP Control Plane (BGPv2) : Cilium Custom Resources 를 통해 BGP 설정 관리 가능 - [Docs](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/)

![[Pasted image 20250810204241.png]]
- `CiliumBGPClusterConfig`: Defines BGP instances and peer configurations that are applied to multiple nodes.
- `CiliumBGPPeerConfig`: A common set of BGP peering setting. It can be used across multiple peers.
- `CiliumBGPAdvertisement`: Defines prefixes that are injected into the BGP routing table.
- `CiliumBGPNodeConfigOverride`: Defines node-specific BGP configuration to provide a finer control.

# BGP 설정 후 통신 확인 : Cilium의 BGP는 기본적으로 외부 경로를 커널 라우팅 테이블에 주입하지 않음.

router 접속 후 설정 : sshpass -p 'vagrant' ssh vagrant@router
```bash
# k8s-ctr 에서 router 로 ssh 접속
sshpass -p 'vagrant' ssh vagrant@router

#
ss -tnlp | grep -iE 'zebra|bgpd'
ps -ef |grep frr
root        4127       1  0 13:38 ?        00:00:00 /usr/lib/frr/watchfrr -d -F traditional zebra bgpd staticd
frr         4140       1  0 13:38 ?        00:00:00 /usr/lib/frr/zebra -d -F traditional -A 127.0.0.1 -s 90000000
frr         4145       1  0 13:38 ?        00:00:00 /usr/lib/frr/bgpd -d -F traditional -A 127.0.0.1
frr         4152       1  0 13:38 ?        00:00:00 /usr/lib/frr/staticd -d -F traditional -A 127.0.0.1

#
vtysh -c 'show running'
cat /etc/frr/frr.conf 
...
log syslog informational
!
router bgp 65000
  bgp router-id 192.168.10.200
  bgp graceful-restart
  no bgp ebgp-requires-policy
  bgp bestpath as-path multipath-relax
  maximum-paths 4

#
vtysh -c 'show running'
vtysh -c 'show ip bgp summary'
vtysh -c 'show ip bgp'
BGP table version is 1, local router ID is 192.168.10.200, vrf id 0
Default local pref 100, local AS 65000
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.10.1.0/24     0.0.0.0                  0         32768 i

```

```bash
ip -c route
vtysh -c 'show ip route'
...
K>* 0.0.0.0/0 [0/100] via 10.0.2.2, eth0, src 10.0.2.15, 00:34:45
...
C>* 192.168.10.0/24 is directly connected, eth1, 00:34:45
C>* 192.168.20.0/24 is directly connected, eth2, 00:34:45

# Cilium node 연동 설정 방안 1
cat << EOF >> /etc/frr/frr.conf
  neighbor CILIUM peer-group
  neighbor CILIUM remote-as external
  neighbor 192.168.10.100 peer-group CILIUM
  neighbor 192.168.10.101 peer-group CILIUM
  neighbor 192.168.20.100 peer-group CILIUM 
EOF

```

```bash
cat /etc/frr/frr.conf

systemctl daemon-reexec && systemctl restart frr
systemctl status frr --no-pager --full

# 모니터링 걸어두기!
journalctl -u frr -f


# Cilium node 연동 설정 방안 2
vtysh
---------------------------
?
show ?
show running
show ip route

# config 모드 진입
conf
?

## bgp 65000 설정 진입
router bgp 65000
?
neighbor CILIUM peer-group
neighbor CILIUM remote-as external
neighbor 192.168.10.100 peer-group CILIUM
neighbor 192.168.10.101 peer-group CILIUM
neighbor 192.168.20.100 peer-group CILIUM 
end

# Write configuration to the file (same as write file)
write memory

exit
---------------------------

cat /etc/frr/frr.conf

```

# cilium 에 bgp 설정
```bash
# 신규 터미널 1 (router) : 모니터링 걸어두기!
journalctl -u frr -f

# 신규 터미널 2 (k8s-ctr) : 반복 호출
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s --connect-timeout 1 webpod | grep Hostname; echo "---" ; sleep 1; done'


# BGP 동작할 노드를 위한 label 설정
kubectl label nodes k8s-ctr k8s-w0 k8s-w1 enable-bgp=true
kubectl get node -l enable-bgp=true


# Config Cilium BGP
cat << EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumBGPAdvertisement
metadata:
  name: bgp-advertisements
  labels:
    advertise: bgp
spec:
  advertisements:
    - advertisementType: "PodCIDR"
---
apiVersion: cilium.io/v2
kind: CiliumBGPPeerConfig
metadata:
  name: cilium-peer
spec:
  timers:
    holdTimeSeconds: 9
    keepAliveTimeSeconds: 3
  ebgpMultihop: 2
  gracefulRestart:
    enabled: true
    restartTimeSeconds: 15
  families:
    - afi: ipv4
      safi: unicast
      advertisements:
        matchLabels:
          advertise: "bgp"
---
apiVersion: cilium.io/v2
kind: CiliumBGPClusterConfig
metadata:
  name: cilium-bgp
spec:
  nodeSelector:
    matchLabels:
      "enable-bgp": "true"
  bgpInstances:
  - name: "instance-65001"
    localASN: 65001
    peers:
    - name: "tor-switch"
      peerASN: 65000
      peerAddress: 192.168.10.200  # router ip address
      peerConfigRef:
        name: "cilium-peer"
EOF
```

# 통신 확인
```bash
# BGP 연결 확인
ss -tnlp | grep 179
ss -tnp | grep 179

# cilium bgp 정보 확인
cilium bgp peers
cilium bgp routes available ipv4 unicast

kubectl get ciliumbgpadvertisements,ciliumbgppeerconfigs,ciliumbgpclusterconfigs
kubectl get ciliumbgpnodeconfigs -o yaml | yq
...
                "peeringState": "established",
                "routeCount": [
                  {
                    "advertised": 2,
                    "afi": "ipv4",
                    "received": 1,
                    "safi": "unicast"
                  }
...


# 신규 터미널 1 (router) : 모니터링 걸어두기!
journalctl -u frr -f
Aug 09 14:31:40 router bgpd[4665]: [M59KS-A3ZXZ] bgp_update_receive: rcvd End-of-RIB for IPv4 Unicast from 192.168.20.100 in vrf default
Aug 09 14:31:40 router bgpd[4665]: [M59KS-A3ZXZ] bgp_update_receive: rcvd End-of-RIB for IPv4 Unicast from 192.168.10.101 in vrf default
Aug 09 14:31:40 router bgpd[4665]: [M59KS-A3ZXZ] bgp_update_receive: rcvd End-of-RIB for IPv4 Unicast from 192.168.10.100 in vrf default

ip -c route | grep bgp
172.20.0.0/24 nhid 32 via 192.168.10.100 dev eth1 proto bgp metric 20
172.20.1.0/24 nhid 30 via 192.168.10.101 dev eth1 proto bgp metric 20
172.20.2.0/24 nhid 31 via 192.168.20.100 dev eth2 proto bgp metric 20

vtysh -c 'show ip bgp summary'
Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
192.168.10.100  4      65001       509       511        0    0    0 00:25:15            1        4 N/A
192.168.10.101  4      65001       508       511        0    0    0 00:25:15            1        4 N/A
192.168.20.100  4      65001       509       511        0    0    0 00:25:15            1        4 N/A

vtysh -c 'show ip bgp'
   Network          Next Hop            Metric LocPrf Weight Path
*> 10.10.1.0/24     0.0.0.0                  0         32768 i
*> 172.20.0.0/24    192.168.10.100                         0 65001 i
*> 172.20.1.0/24    192.168.10.101                         0 65001 i
*> 172.20.2.0/24    192.168.20.100                         0 65001 i


# 신규 터미널 2 (k8s-ctr) : 반복 호출???
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s --connect-timeout 1 webpod | grep Hostname; echo "---" ; sleep 1; done'
```


# BGP 정보 전달 확인
![[Pasted image 20250810204902.png]]

```bash
# k8s-ctr tcpdump 해두기
tcpdump -i eth1 tcp port 179 -w /tmp/bgp.pcap

# router : frr 재시작
systemctl restart frr && journalctl -u frr -f


# bgp.type == 2
termshark -r /tmp/bgp.pcap
 
# 분명 Router 장비를 통해 BGP UPDATE로 받음을 확인.
cilium bgp routes
ip -c route
```

# Cilium BGP는 외부 경로를 커널 라우팅 테이블에 주입하지 않음
- ChatGPT 물어본 내용 : Cilium의 BGP는 기본적으로 외부 경로를 커널 라우팅 테이블에 주입하지 않음.
    - 왜 Cilium이 받은 BGP 경로가 K8s 노드 OS 커널 라우팅 테이블에 안 들어오나?
        - Cilium의 BGP는 "컨트롤 플레인"만 동작
            - Cilium BGP Speaker(GoBGP 기반)는 BGP 세션을 맺고 prefix를 광고하거나 수신합니다.
            - 하지만 수신한 경로를 Linux 커널(FIB) 에 바로 주입하지 않음.
            - 대신 Cilium 내부에서 LoadBalancer 서비스 광고, PodCIDR 전파 같은 용도로만 사용.
        - Pod/Service 네트워크 경로는 Cilium eBPF가 처리
            - Cilium은 kube-proxy 대체 모드에서 eBPF datapath로 패킷을 라우팅합니다.
            - 외부 경로 학습이 커널 라우팅 테이블에 없어도, eBPF map에 저장된 다음 홉 정보로 처리 가능.
        - GoBGP 기본 설정도 FIB 설치 비활성화
            - Cilium이 사용하는 GoBGP 라이브러리는 `disable-telemetry`, `disable-fib` 상태로 빌드됨.
            - 즉, 외부 라우터에서 들어온 BGP NLRI는 커널에 반영되지 않고, Cilium 내부 정책/광고 로직에서만 사용.


# 문제 해결 후 통신 확인 : 결론은 Cilium 으로 BGP 사용 시, 2개 이상의 NIC 사용할 경우에는 Node에 직접 라우팅 설정 및 관리가 필요함.

- 현재 실습 환경은 2개의 NIC(eth0, eth1)을 사용하고 있는 상황으로, default GW가 eth0 경로로 설정 되어 있음.
- eth1은 k8s 통신 용도로 사용 중. 즉, 현재 k8s 파드 사용 대역 통신 전체는 eth1을 통해서 라우팅 설정하면됨.
- 해당 라우팅을 상단에 네트워크 장비가 받게 되고, 해당 장비는 Cilium Node를 통해 모든 PodCIDR 정보를 알고 있기에, 목적지로 전달 가능함.
- 결론은 Cilium 으로 BGP 사용 시, 2개 이상의 NIC 사용할 경우에는 Node에 직접 라우팅 설정 및 관리가 필요함.

```bash
# k8s 파드 사용 대역 통신 전체는 eth1을 통해서 라우팅 설정
ip route add 172.20.0.0/16 via 192.168.10.200
sshpass -p 'vagrant' ssh vagrant@k8s-w1 sudo ip route add 172.20.0.0/16 via 192.168.10.200
sshpass -p 'vagrant' ssh vagrant@k8s-w0 sudo ip route add 172.20.0.0/16 via 192.168.20.200

# router 가 bgp로 학습한 라우팅 정보 한번 더 확인 : 
sshpass -p 'vagrant' ssh vagrant@router ip -c route | grep bgp
172.20.0.0/24 nhid 64 via 192.168.10.100 dev eth1 proto bgp metric 20 
172.20.1.0/24 nhid 60 via 192.168.10.101 dev eth1 proto bgp metric 20 
172.20.2.0/24 nhid 62 via 192.168.20.100 dev eth2 proto bgp metric 20 

# 정상 통신 확인!
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s --connect-timeout 1 webpod | grep Hostname; echo "---" ; sleep 1; done'

# hubble relay 포트 포워딩 실행
cilium hubble port-forward&
hubble status

# flow log 모니터링
hubble observe -f --protocol tcp --pod curl-pod
```


# 노드 유지보수 (k8s-w0) 시 - [Docs](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-operation/#shutting-down-a-node)
노드 유지보수를 위한 설정
```bash
# 모니터링 : 반복 호출
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s --connect-timeout 1 webpod | grep Hostname; echo "---" ; sleep 1; done'

# (참고) BGP Control Plane logs
kubectl logs -n kube-system -l name=cilium-operator -f | grep "subsys=bgp-cp-operator"
kubectl logs -n kube-system -l k8s-app=cilium -f | grep "subsys=bgp-control-plane"


# 유지보수를 위한 설정
kubectl drain k8s-w0 --ignore-daemonsets
kubectl label nodes k8s-w0 enable-bgp=false --overwrite

# 확인
kubectl get node
kubectl get ciliumbgpnodeconfigs
cilium bgp routes
cilium bgp peers
Node      Local AS   Peer AS   Peer Address     Session State   Uptime     Family         Received   Advertised
k8s-ctr   65001      65000     192.168.10.200   established     2h13m35s   ipv4/unicast   3          2    
k8s-w1    65001      65000     192.168.10.200   established     2h13m36s   ipv4/unicast   3          2   

sshpass -p 'vagrant' ssh vagrant@router "sudo vtysh -c 'show ip bgp summary'"
sshpass -p 'vagrant' ssh vagrant@router "sudo vtysh -c 'show ip bgp'"
sshpass -p 'vagrant' ssh vagrant@router "sudo vtysh -c 'show ip route bgp'"
sshpass -p 'vagrant' ssh vagrant@router ip -c route | grep bgp
172.20.0.0/24 nhid 64 via 192.168.10.100 dev eth1 proto bgp metric 20 
172.20.1.0/24 nhid 60 via 192.168.10.101 dev eth1 proto bgp metric 20 
```

# 원복 설정
```bash
# 원복 설정
kubectl label nodes k8s-w0 enable-bgp=true --overwrite
kubectl uncordon k8s-w0

# 확인
kubectl get node
kubectl get ciliumbgpnodeconfigs
cilium bgp routes
cilium bgp peers

sshpass -p 'vagrant' ssh vagrant@router "sudo vtysh -c 'show ip bgp summary'"
sshpass -p 'vagrant' ssh vagrant@router "sudo vtysh -c 'show ip bgp'"
sshpass -p 'vagrant' ssh vagrant@router "sudo vtysh -c 'show ip route bgp'"
sshpass -p 'vagrant' ssh vagrant@router ip -c route | grep bgp

# 노드별 파드 분배 실행
kubectl get pod -owide
kubectl scale deployment webpod --replicas 0
kubectl scale deployment webpod --replicas 3
```

# 도전과제1 Descheduler for Kubernetes : Pod의 상태를 확인하여 조건에 성립하는 Pod를 Eviction하여 원하는 상태로 만듬 - [Github](https://github.com/kubernetes-sigs/descheduler) , [Blog](https://ybchoi.com/19)

^e71ff2

```bash
# Run As A Deployment
kubectl kustomize 'https://github.com/kubernetes-sigs/descheduler.git/kubernetes/deployment?ref=release-1.33' | kubectl apply -f -
serviceaccount/descheduler-sa created
clusterrole.rbac.authorization.k8s.io/descheduler-cluster-role created
clusterrolebinding.rbac.authorization.k8s.io/descheduler-cluster-role-binding created
configmap/descheduler-policy-configmap created
deployment.apps/descheduler created

#
kubectl get deploy -n kube-system descheduler
kubectl get cm -n kube-system descheduler-policy-configmap

kubectl describe pod -n kube-system -l app=descheduler
kubectl describe cm -n kube-system descheduler-policy-configmap
...
policy.yaml:
----
apiVersion: "descheduler/v1alpha2"
kind: "DeschedulerPolicy"
profiles:
  - name: ProfileName
    pluginConfig:
    - name: "DefaultEvictor"
    - name: "RemovePodsViolatingInterPodAntiAffinity"
    - name: "RemoveDuplicates"
    - name: "LowNodeUtilization"
      args:
        thresholds:
          "cpu" : 20
          "memory": 20
          "pods": 20
        targetThresholds:
          "cpu" : 50
          "memory": 50
          "pods": 50
          
    plugins:
      balance:
        enabled:
          - "LowNodeUtilization"
          - "RemoveDuplicates"
      deschedule:
        enabled:
          - "RemovePodsViolatingInterPodAntiAffinity"
```
![[Pasted image 20250810211425.png]]

# Disabling CRD Status Report : 노드가 많은 대규모 클러스터의 경우, api 서버에 부하 유발할 수 있으니, bgp status reporting off 권장 - [Docs](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-operation/#disabling-crd-status-report)
```bash
# 확인
kubectl get ciliumbgpnodeconfigs -o yaml | yq

# 설정
helm upgrade cilium cilium/cilium --version 1.18.0 --namespace kube-system --reuse-values \
  --set bgpControlPlane.statusReport.enabled=false

kubectl -n kube-system rollout restart ds/cilium


# 확인 : CiliumBGPNodeConfig Status 정보가 없다!
kubectl get ciliumbgpnodeconfigs -o yaml | yq
...
      "status": {}
```

# Service(LoadBalancer - ExternalIP) IPs 를 BGP로 광고 - [Docs](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/#service-virtual-ips)
![[Pasted image 20250810211924.png]]
```bash
# LB IPAM Announcement over BGP 설정 예정으로, 노드의 네트워크 대역이 아니여도 가능!
cat << EOF | kubectl apply -f -
apiVersion: "cilium.io/v2"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "cilium-pool"
spec:
  allowFirstLastIPs: "No"
  blocks:
  - cidr: "172.16.1.0/24"
EOF
```

```bash
kubectl get ippool
NAME          DISABLED   CONFLICTING   IPS AVAILABLE   AGE
cilium-pool   false      False         254             8s


#
kubectl patch svc webpod -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc webpod 
NAME     TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
webpod   LoadBalancer   10.96.39.92   172.16.1.1    80:30800/TCP   3h56m

kubectl get ippool
NAME          DISABLED   CONFLICTING   IPS AVAILABLE   AGE
cilium-pool   false      False         253             2m23s

kubectl describe svc webpod | grep 'Traffic Policy'
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster

kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg service list


# LBIP로 curl 요청 확인
kubectl get svc webpod -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
LBIP=$(kubectl get svc webpod -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s $LBIP
curl -s $LBIP | grep Hostname
curl -s $LBIP | grep RemoteAddr


# 모니터링
watch "sshpass -p 'vagrant' ssh vagrant@router ip -c route"


# LB EX-IP를 BGP로 광고 설정
cat << EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumBGPAdvertisement
metadata:
  name: bgp-advertisements-lb-exip-webpod
  labels:
    advertise: bgp
spec:
  advertisements:
    - advertisementType: "Service"
      service:
        addresses:
          - LoadBalancerIP
      selector:             
        matchExpressions:
          - { key: app, operator: In, values: [ webpod ] }
EOF
```

```bash
kubectl get CiliumBGPAdvertisement
NAME                                AGE
bgp-advertisements                  2m1s
bgp-advertisements-lb-exip-webpod   3s


# 확인
kubectl exec -it -n kube-system ds/cilium -- cilium-dbg bgp route-policies
VRouter   Policy Name                                             Type     Match Peers         Match Families   Match Prefixes (Min..Max Len)   RIB Action   Path Actions
65001     allow-local                                             import                                                                        accept
65001     tor-switch-ipv4-PodCIDR                                 export   192.168.10.200/32                    172.20.1.0/24 (24..24)          accept
65001     tor-switch-ipv4-Service-webpod-default-LoadBalancerIP   export   192.168.10.200/32                    172.16.1.1/32 (32..32)          accept

cilium bgp routes available ipv4 unicast
Node      VRouter   Prefix          NextHop   Age      Attrs
k8s-ctr   65001     172.16.1.1/32   0.0.0.0   32s      [{Origin: i} {Nexthop: 0.0.0.0}]   
          65001     172.20.0.0/24   0.0.0.0   24m41s   [{Origin: i} {Nexthop: 0.0.0.0}]   
k8s-w0    65001     172.16.1.1/32   0.0.0.0   32s      [{Origin: i} {Nexthop: 0.0.0.0}]   
          65001     172.20.2.0/24   0.0.0.0   24m56s   [{Origin: i} {Nexthop: 0.0.0.0}]   
k8s-w1    65001     172.16.1.1/32   0.0.0.0   32s      [{Origin: i} {Nexthop: 0.0.0.0}]   
          65001     172.20.1.0/24   0.0.0.0   24m56s   [{Origin: i} {Nexthop: 0.0.0.0}]  

# 현재 BGP가 동작하는 모든 노드로 전달 가능!
sshpass -p 'vagrant' ssh vagrant@router ip -c route
...
172.16.1.1 nhid 71 proto bgp metric 20 
        nexthop via 192.168.10.101 dev eth1 weight 1 
        nexthop via 192.168.10.100 dev eth1 weight 1 
        nexthop via 192.168.20.100 dev eth2 weight 1 

sshpass -p 'vagrant' ssh vagrant@router "sudo vtysh -c 'show ip route bgp'"
sshpass -p 'vagrant' ssh vagrant@router "sudo vtysh -c 'show ip bgp summary'"
sshpass -p 'vagrant' ssh vagrant@router "sudo vtysh -c 'show ip bgp'"
   Network          Next Hop            Metric LocPrf Weight Path    # * valid, > best, = multipath
*> 172.16.1.1/32    192.168.10.100                         0 65001 i 
*=                  192.168.20.100                         0 65001 i 
*=                  192.168.10.101                         0 65001 i

sshpass -p 'vagrant' ssh vagrant@router "sudo vtysh -c 'show ip bgp 172.16.1.1/32'"
BGP routing table entry for 172.16.1.1/32, version 7
Paths: (3 available, best #1, table default)
  Advertised to non peer-group peers:
  192.168.10.100 192.168.10.101 192.168.20.100
  65001
    192.168.10.100 from 192.168.10.100 (192.168.10.100)
      Origin IGP, valid, external, multipath, best (Router ID)
      Last update: Sat Aug  9 17:50:29 2025
  65001
    192.168.20.100 from 192.168.20.100 (192.168.20.100)
      Origin IGP, valid, external, multipath
      Last update: Sat Aug  9 17:50:29 2025
  65001
    192.168.10.101 from 192.168.10.101 (192.168.10.101)
      Origin IGP, valid, external, multipath
      Last update: Sat Aug  9 17:50:29 2025

```

# router 에서 LB EX-IP 호출 확인
```bash 
#
LBIP=172.16.1.1
curl -s $LBIP
curl -s $LBIP | grep Hostname
curl -s $LBIP | grep RemoteAddr

# 반복 접속
for i in {1..100};  do curl -s $LBIP | grep Hostname; done | sort | uniq -c | sort -nr
while true; do curl -s $LBIP | egrep 'Hostname|RemoteAddr' ; sleep 0.1; done


# k8s-ctr 에서 replicas=2 로 줄여보자
kubectl scale deployment webpod --replicas 2
kubectl get pod -owide
cilium bgp routes


# router 에서 정보 확인 : k8s-ctr 노드에 대상 파드가 배치되지 않았지만, 라우팅 경로 설정이 되어 있다.
ip -c route
vtysh -c 'show ip bgp summary'
vtysh -c 'show ip bgp'
vtysh -c 'show ip bgp 172.16.1.1/32'
vtysh -c 'show ip route bgp'

# 반복 접속 : ??? RemoteAddr 이 왜 10.100???
for i in {1..100};  do curl -s $LBIP | grep Hostname; done | sort | uniq -c | sort -nr
while true; do curl -s $LBIP | egrep 'Hostname|RemoteAddr' ; sleep 0.1; done
Hostname: webpod-697b545f57-swtdz
RemoteAddr: 192.168.10.100:40460
Hostname: webpod-697b545f57-87lf2
RemoteAddr: 192.168.10.100:40474
...


# 신규터미널 (3개) : k8s-w1, k8s-w2, k8s-w0
tcpdump -i eth1 -A -s 0 -nn 'tcp port 80'

# k8s-ctr 를 경유하거나 등 확인 : ExternalTrafficPolicy 설정 확인
LBIP=172.16.1.1
curl -s $LBIP
curl -s $LBIP
curl -s $LBIP
curl -s $LBIP
```

# (정보) ECMP (equal cost multipath) is a method to utilize multiple same-cost paths to route a packet to a destination - [Blog](https://createnetech.tistory.com/51)

![[Pasted image 20250810212123.png]]
* 실제 라우팅 처리 되는 부하 분산 알고리즘은 해당 HW/SW 다를 수 있음. 예) 목적지 IP/Port 다를 경우 다를 경로로 전달, per-packer 별 등등.

# External Traffic Policy (Local) : 소스 IP 보존 - [Youtube](https://www.youtube.com/watch?v=Tv0R6VxyWhc) , [Docs](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/#externaltrafficpolicy-internaltrafficpolicy) & Linux ECMP Hash Policy
![[Pasted image 20250810212222.png]]

```bash
# 모니터링 
watch "sshpass -p 'vagrant' ssh vagrant@router ip -c route"


# k8s-ctr
kubectl patch service webpod -p '{"spec":{"externalTrafficPolicy":"Local"}}'


# router(frr) : 서비스에 대상 파드가 배치된 노드만 BGP 경로에 출력!
vtysh -c 'show ip bgp'
vtysh -c 'show ip bgp 172.16.1.1/32'
vtysh -c 'show ip route bgp'
ip -c route


# 신규터미널 (3개) : k8s-w1, k8s-w2, k8s-w0
tcpdump -i eth1 -A -s 0 -nn 'tcp port 80'


# 현재 실습 환경 경우 반복 접속 시 한쪽 노드로 선택되고, 소스IP가 보존!
LBIP=172.16.1.1
curl -s $LBIP
for i in {1..100};  do curl -s $LBIP | grep Hostname; done | sort | uniq -c | sort -nr
while true; do curl -s $LBIP | egrep 'Hostname|RemoteAddr' ; sleep 0.1; done

## 아래 실행 시 tcpdump 에 다른 노드 선택되는지 확인! 안될수도 있음!
curl -s $LBIP --interface 10.10.1.200
curl -s $LBIP --interface 10.10.2.200
```
## Linux ECMP Hash Policy 설정
```bash
# 리눅스 커널은 기본적으로 L3(목적지 IP 기반) 해시를 사용합니다. 보다 정교한 부하분산을 원하면 L4 해시 (IP + 포트) 기반으로 설정
# 1 : source IP, dest IP, source port, dest port 기반 hash (more granular)로 변경
sudo sysctl -w net.ipv4.fib_multipath_hash_policy=1
echo "net.ipv4.fib_multipath_hash_policy=1" >> /etc/sysctl.conf

# 
for i in {1..100};  do curl -s $LBIP | grep Hostname; done | sort | uniq -c | sort -nr
     59 Hostname: webpod-697b545f57-87lf2
     41 Hostname: webpod-697b545f57-swtdz
    

# k8s-ctr
kubectl scale deployment webpod --replicas 3
kubectl get pod -owide

# router
ip -c route
for i in {1..100};  do curl -s $LBIP | grep Hostname; done | sort | uniq -c | sort -nr
     37 Hostname: webpod-697b545f57-bgpv9
     35 Hostname: webpod-697b545f57-87lf2
     28 Hostname: webpod-697b545f57-swtdz
```

ExternalTrafficPolicy(Local) 설정 시, Router 의 ECMP 에서 Hash Policy 경로 결정과 요청 트래픽 환경(소스 IP, 포트 등)으로 특정 동일 노드로만 라우팅 될 수 있음 → 대체로 Hash Policy 를 L4 수준 설정 권장.



# (중급) 인입 시 다양한 방식 비교 + 도전과제5 포함
- BGP(ECMP) + Service(LB EX-IP, ExternalTrafficPolicy:Local) + SNAT + Random vs.
- BGP(ECMP) + Service(LB EX-IP, ExternalTrafficPolicy:Cluster) + DSR + Maglev 2가지 외부 인입 방식 장단점(성능, Failover 등) 비교
- `도전과제5` L2 Announcement + Service(LB EX-IP, ExternalTrafficPolicy:Cluster) + DSR + Maglev 설정 후 인입 과정 분석 ^e3e50a
## BGP(ECMP) + Service(LB EX-IP, ExternalTrafficPolicy:Local) + SNAT + Random → 권장 방식!

![[Pasted image 20250810212444.png]]

(1) Router 에서 BGP ECMP MultiPath로 파드가 있는 노드로만 전달
(2) 해당 Service 가 ExternalTrafficPolicy:Local 로, 소스IP 보존되고 대상 파드가 응답 후 Router로 바로 리턴

## BGP(ECMP) + Service(LB EX-IP, ExternalTrafficPolicy:Cluster) + SNAT → 비권장 방식
![[Pasted image 20250810212628.png]]
(1) Router 에서 BGP ECMP MultiPath로 Cilium BGP Peer인 모든 노드로 전달
(2) 해당 Service 가 ExternalTrafficPolicy:Cluster로, 하필 다른 노드(k8s-ctr)의 파드로 요청을 전달
(3) 해당 노드의 파드가 요청을 처리하고 응답 리턴을 위해서, NAT를 수행했던 노드(k8s-w1)로 다시 전달
(4) 외부 인입을 받아서 NAT를 수행했던 연결 정보를 확인해서, Reverse NAT를 수행해서 최종 응답을 리턴

## BGP(ECMP) + Service(LB EX-IP, ExternalTrafficPolicy:Cluster) + DSR + Maglev - [Docs](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#direct-server-return-dsr) → 그나마 괜찮은 방식
![[Pasted image 20250810212706.png]]
(1) Router 에서 BGP ECMP MultiPath로 Cilium BGP Peer인 모든 노드로 전달
(2) 해당 Service 가 ExternalTrafficPolicy:Cluster로, 하필 다른 노드(k8s-ctr)의 파드로 요청을 전달
- 이때 DSR 동작으로 전달된 노드에서 바로 리턴하기 위해서, 최초 접속했던 클라이언트의 정보를 GENEVE 헤더를 활용해서 전달
- 장애 및 백엔드 대상 변경에서 클라이언트의 세션에 대한 똑똑한 유지를 위해서 Maglev 알고리즘을 사용
(3) 해당 노드의 파드가 요청을 처리하고 GEVEVE 헤더 정보를 활용하여 Reverse NAT를 수행해서 최종 응답을 바로 리턴

```bash
# 현재 설정 확인
kubectl exec -it -n kube-system ds/cilium -- cilium status --verbose
...
  Mode:                 SNAT
  Backend Selection:    Random
  Session Affinity:     Enabled
...

# 현재 실습 환경에서 설정
modprobe geneve # modprobe geneve
lsmod | grep -E 'vxlan|geneve'
for i in w1 w0 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo modprobe geneve ; echo; done
for i in w1 w0 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo lsmod | grep -E 'vxlan|geneve' ; echo; done

helm upgrade cilium cilium/cilium --version 1.18.0 --namespace kube-system --reuse-values \
  --set tunnelProtocol=geneve --set loadBalancer.mode=dsr --set loadBalancer.dsrDispatch=geneve \
  --set loadBalancer.algorithm=maglev

kubectl -n kube-system rollout restart ds/cilium

# 설정 확인
kubectl exec -it -n kube-system ds/cilium -- cilium status --verbose
...
  Mode:                  DSR
    DSR Dispatch Mode:   Geneve
  Backend Selection:     Maglev (Table Size: 16381)
  Session Affinity:     Enabled
...

# Cluster 로 기본 설정 원복
kubectl patch svc webpod -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'

# k8s-ctr, k8s-w1, k8s-w0 모두 tcpdump 실행
tcpdump -i eth1 -w /tmp/dsr.pcap

# router
curl -s $LBIP
curl -s $LBIP
curl -s $LBIP
curl -s $LBIP
curl -s $LBIP

 
# Host 에 pcap 다운로드 후 wireshark 로 열여서 확인
vagrant plugin install vagrant-scp
vagrant scp k8s-ctr:/tmp/dsr.pcap .

```
### 최초 192.168.10.101(k8s-w1)로 인입 후 하필 10.101(k8s-ctr)에 전달. 전달 시 최초 클라이언트 접속 정보를 GENEVE 헤더에 담아서 전달
![[Pasted image 20250810212859.png]]
### 아래 k8s-ctr 에서 바로 외부 클라이언트(router)로 리턴하는 패킷 정보 확인. D.IP 와 D.Port 가 최초 요청 시 S.IP, S.Port 와 동일(유지)를 확인
![[Pasted image 20250810212928.png]]

# (고급) Service 추가 후 Node 별 분산 인입 설정
- 목표
    - Service1 LB EX-IP 는 az1=true 로 인입 가능
    - Service2 LB EX-IP 는 az2=true 로 인입 가능
    - AZ=1 소속 : k8s-ctr, k8s-w1
    - AZ=2 소속 : k8s-w0
## 실습 환경 마련 : node label
```bash
# node 별 label 설정
kubectl label nodes k8s-ctr k8s-w1 az1=true
kubectl label nodes k8s-w0         az2=true

# 확인
kubectl get node -l az1=true
kubectl get node -l az2=true
```

## 실습 환경 마련 : Service 추가
```bash
#
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netshoot-web
  labels:
    app: netshoot-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: netshoot-web
  template:
    metadata:
      labels:
        app: netshoot-web
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: netshoot
          image: nicolaka/netshoot
          ports:
            - containerPort: 8080
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          command: ["sh", "-c"]
          args:
            - |
              while true; do 
                { echo -e "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\nOK from \$POD_NAME"; } | nc -l -p 8080 -q 1;
              done
EOF
```
```bash
#
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: netshoot-web
  labels:
    app: netshoot-web
spec:
  type: LoadBalancer
  selector:
    app: netshoot-web
  ports:
    - name: http
      port: 80      
      targetPort: 8080
EOF
```
```bash
#
kubectl get svc,ep netshoot-web
kubectl get ippool

```

## Cilium BGP 설정 : Service 1
```bash
# 모니터링 (router)
watch "vtysh -c 'show ip route bgp'"


# 기존 BGP 설정 제거
kubectl delete ciliumbgpadvertisements,ciliumbgppeerconfigs,ciliumbgpclusterconfigs --all


# Service 1 광고를 위한 Cilium BGP
cat << EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumBGPAdvertisement
metadata:
  name: service1-bgp-advertisements
  labels:
    advertise: service1-bgp
spec:
  advertisements:
    - advertisementType: "PodCIDR"
    - advertisementType: "Service"
      service:
        addresses:
          - LoadBalancerIP
      selector:             
        matchExpressions:
          - { key: az1, operator: In, values: [ "true" ] }
---
apiVersion: cilium.io/v2
kind: CiliumBGPPeerConfig
metadata:
  name: service1-cilium-peer
spec:
  timers:
    holdTimeSeconds: 9
    keepAliveTimeSeconds: 3
  ebgpMultihop: 2
  gracefulRestart:
    enabled: true
    restartTimeSeconds: 15
  families:
    - afi: ipv4
      safi: unicast
      advertisements:
        matchLabels:
          advertise: service1-bgp
---
apiVersion: cilium.io/v2
kind: CiliumBGPClusterConfig
metadata:
  name: service1-cilium-bgp
spec:
  nodeSelector:
    matchLabels:
      "az1": "true"
  bgpInstances:
  - name: "instance-65001"
    localASN: 65001
    peers:
    - name: "tor-switch"
      peerASN: 65000
      peerAddress: 192.168.10.200  # router ip address
      peerConfigRef:
        name: "service1-cilium-peer"
EOF
```
```bash

# 모니터링 (router)
watch "vtysh -c 'show ip route bgp'"
B>* 172.20.0.0/24 [20/0] via 192.168.10.100, eth1, weight 1, 00:00:18
B>* 172.20.1.0/24 [20/0] via 192.168.10.101, eth1, weight 1, 00:00:18


# Service 1 광고를 위한 label 설정
kubectl label service webpod az1=true


# 모니터링 (router)
watch "vtysh -c 'show ip route bgp'"
B>* 172.16.1.1/32 [20/0] via 192.168.10.100, eth1, weight 1, 00:01:00
  *                      via 192.168.10.101, eth1, weight 1, 00:01:00
B>* 172.20.0.0/24 [20/0] via 192.168.10.100, eth1, weight 1, 00:00:18
B>* 172.20.1.0/24 [20/0] via 192.168.10.101, eth1, weight 1, 00:00:18

```

## Cilium BGP 설정 : Service 2
```bash
# 모니터링 (router)
watch "vtysh -c 'show ip route bgp'"


# Service 2 광고를 위한 Cilium BGP
cat << EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumBGPAdvertisement
metadata:
  name: service2-bgp-advertisements
  labels:
    advertise: service2-bgp
spec:
  advertisements:
    - advertisementType: "PodCIDR"
    - advertisementType: "Service"
      service:
        addresses:
          - LoadBalancerIP
      selector:             
        matchExpressions:
          - { key: az2, operator: In, values: [ "true" ] }
---
apiVersion: cilium.io/v2
kind: CiliumBGPPeerConfig
metadata:
  name: service2-cilium-peer
spec:
  timers:
    holdTimeSeconds: 9
    keepAliveTimeSeconds: 3
  ebgpMultihop: 2
  gracefulRestart:
    enabled: true
    restartTimeSeconds: 15
  families:
    - afi: ipv4
      safi: unicast
      advertisements:
        matchLabels:
          advertise: service2-bgp
---
apiVersion: cilium.io/v2
kind: CiliumBGPClusterConfig
metadata:
  name: service2-cilium-bgp
spec:
  nodeSelector:
    matchLabels:
      "az2": "true"
  bgpInstances:
  - name: "instance-65001"
    localASN: 65001
    peers:
    - name: "tor-switch"
      peerASN: 65000
      peerAddress: 192.168.10.200  # router ip address
      peerConfigRef:
        name: "service2-cilium-peer"
EOF
```

```bash
# 모니터링 (router)
watch "vtysh -c 'show ip route bgp'"
B>* 172.16.1.1/32 [20/0] via 192.168.10.100, eth1, weight 1, 00:01:00
  *                      via 192.168.10.101, eth1, weight 1, 00:01:00
B>* 172.20.0.0/24 [20/0] via 192.168.10.100, eth1, weight 1, 00:00:18
B>* 172.20.1.0/24 [20/0] via 192.168.10.101, eth1, weight 1, 00:00:18
B>* 172.20.2.0/24 [20/0] via 192.168.20.100, eth2, weight 1, 00:00:05


# Service 2 광고를 위한 label 설정
kubectl label service netshoot-web az2=true


# 모니터링 (router)
watch "vtysh -c 'show ip route bgp'"
B>* 172.16.1.1/32 [20/0] via 192.168.10.100, eth1, weight 1, 00:01:00
  *                      via 192.168.10.101, eth1, weight 1, 00:01:00
B>* 172.16.1.2/32 [20/0] via 192.168.20.100, eth2, weight 1, 00:04:31
B>* 172.20.0.0/24 [20/0] via 192.168.10.100, eth1, weight 1, 00:00:18
B>* 172.20.1.0/24 [20/0] via 192.168.10.101, eth1, weight 1, 00:00:18
B>* 172.20.2.0/24 [20/0] via 192.168.20.100, eth2, weight 1, 00:00:05


# 만약 externalTrafficPolicy(local) 설정 상태에서, replicas 1로 축소 시, 해당 파드가 k8s-ctr, k8s-w1 에 있을 경우 어떻게 되는가?
kubectl patch service netshoot-web -p '{"spec":{"externalTrafficPolicy":"Local"}}'
kubectl scale deployment netshoot-web --replicas 1

# 모니터링 (router)
watch "vtysh -c 'show ip route bgp'"
...

-------------------------------------

# k8s-w1 node 에 label 추가 설정 시 어떻게 될까?
kubectl label nodes k8s-w1 az2=true

# 확인
kubectl get node -l az1=true
kubectl get node -l az2=true

# 모니터링 (router)
watch "vtysh -c 'show ip route bgp'"
...


# k8s-w1 node 에 az1 label 제거하면 어떻게 될까?
kubectl label nodes k8s-w1 az1-

# 확인
kubectl get node -l az1=true
kubectl get node -l az2=true


# 모니터링 (router)
watch "vtysh -c 'show ip route bgp'"
...
```

- `주의사항` : 노드 별 특정 Service(LB EX-IP) 전파 그룹을 묶어야 함 _→ Service(LB EX-IP) 기준해서 특정 node 를 중복해서 BGP 광고는 현재는 불가능!_
    - 현재 Cilium CR 리소스 연관 관계와 설정의 구조 상, 노드가 동일한 BGP Router 와 연동 시, 단일 CiliumBGPClusterConfig 만 적용이 가능하다.
    - 즉, CiliumBGPClusterConfig Spec에 nodeSelect 로 특정 노드들을 묶을 수 있고, peerConfigRef 를 통해 CiliumBGPPeerConfig 이 적용되며, 다시 advertisements.matchLabels.advertise 를 통해 CiliumBGPAdvertisement 를 적용한다.
    - 이후 CiliumBGPAdvertisement 에 있는 advertisements 를 통해 특정 Service 를 BGP로 광고 할 수 있다.

# 도전과제2 Cilium BGP로 ClusterIP를 광고해보고 통신 확인 해보시기 바랍니다 - [Docs](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/#clusterip)

^31a55f

# `도전과제3` Internal Traffic Policy : Local 설정 시 CusterIP로 호출 시 어떻게 동작하는지 정리해보시기 바랍니다 - [Docs](https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/)

^df858c

# `도전과제4` Cilium 과 FRR 간 BGP 연동 시, 보안을 위해 MD5 Password 설정을 해보시기 바랍니다 - [Docs](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/#bgp-peer-configuration)

^0119ef


# (심화 정보) BGP Auto-Discovery : ToR Switch 와 Server(Cilium) 간 BGP 작업 편의성 설정 기능
![[Pasted image 20250810213254.png]]


## 기존 설정 (예시)
```bash
# ToR Switch : BGP 연동되는 모든 서버의 IP와 BGP AS 입력 필요
## 연동되는 서버가 20개 경우, 20개의 정보 입력 필요
router bgp 65000
 bgp router-id 192.168.10.200
 no bgp ebgp-requires-policy
 bgp graceful-restart
 bgp bestpath as-path multipath-relax
 neighbor 192.168.10.100 remote-as 65001
 neighbor 192.168.10.101 remote-as 65001
 !
 address-family ipv4 unicast
  network 10.10.1.0/24
  maximum-paths 4
 exit-address-family
exit

# Server(Cilium) : BGP 연동되는 Rack 별 ToR Switch1/2의 정보 입력 필요
apiVersion: cilium.io/v2
kind: CiliumBGPClusterConfig
metadata:
  name: cilium-bgp
spec:
  nodeSelector:
    matchLabels:
      bgp-node: enable
  bgpInstances:
  - name: "instance-65001"
    localASN: 65001
    peers:
    - name: "peer-65000-tor1"
      peerASN: 65000
      peerAddress: 192.168.10.200 
      peerConfigRef:
        name: "cilium-peer"
    - name: "peer-65000-tor2"
      peerASN: 65000
      peerAddress: 192.168.10.201 
      peerConfigRef:
        name: "cilium-peer"
```

## 작업 편의성 설정 (예시) : Default Gateway Auto-Discovery - [Docs](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/#auto-discovery) & FRR BGP Listen Range - [Docs](https://docs.frrouting.org/en/latest/bgp.html#clicmd-bgp-listen-range-A.B.C.D-M-X-X-X-X-M-peer-group-PGNAME)

```bash
# ToR Switch : BGP 연동되는 모든 서버의 IP와 BGP AS 입력 필요 없음!
router bgp 65000
  neighbor CILIUM peer-group
  neighbor CILIUM remote-as external
  bgp listen range 192.168.10.0/24 peer-group CILIUM


# Server(Cilium) : BGP 연동되는 Rack 별 ToR Switch1/2의 IP 입력 필요 없음!, BGP AS 정보만 기입하면됨!
## 단, 서버에 default gw 의 IP만 가능함. 만약 Multi-Homing BGP 연결 경우 A-S 로 동작. A-A 동작 하려면 ToR SW IP 직접 입력 필요.
apiVersion: cilium.io/v2
kind: CiliumBGPClusterConfig
metadata:
  name: cilium-bgp
spec:
  nodeSelector:
    matchLabels:
      bgp-node: enable
  bgpInstances:
  - name: "instance-65001"
    localASN: 65001
    peers:
    - name: "tor-switch"
      peerASN: 65000
      autoDiscovery:
        mode: "DefaultGateway"
        defaultGateway:
          addressFamily: ipv4
      peerConfigRef:
        name: "cilium-peer"
```
Use case : Tencent Cloud - [Blog](https://segmentfault.com/a/1190000040269867/en)

