NodeLocalDNS 소개 [https://popappend.tistory.com/142](https://popappend.tistory.com/142)

Using NodeLocal DNSCache in Kubernetes Clusters 소개 - [Docs](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/)
![[Pasted image 20250728004910.png]]
- `NodeLocal DNSCache`는 클러스터 노드에서 DNS 캐싱 에이전트를 DaemonSet으로 실행하여 클러스터 DNS 성능을 향상시킵니다.
- 오늘날의 아키텍처에서 '`ClusterFirst`' DNS 모드의 Pods는 DNS 쿼리를 위해 kube-dns 서비스 IP에 도달합니다.
- 이는 kube-proxy에 의해 추가된 **iptables** 규칙을 통해 kube-dns/CoreDNS 엔드포인트로 변환됩니다.
- 이 새로운 아키텍처를 통해 Pods는 동일한 노드에서 실행되는 DNS 캐싱 에이전트에 도달하여 iptables DNAT 규칙과 연결 추적을 피할 수 있습니다.
- 로컬 캐싱 에이전트는 클러스터 호스트 이름(기본적으로 “cluster.local” 접미사)의 캐시 누락에 대해 kube-dns 서비스에 쿼리합니다.
- 현재 DNS 아키텍처에서는 로컬 kube-dns/CoreDNS 인스턴스가 없는 경우 DNS QPS가 가장 높은 포드가 다른 노드에 도달해야 할 수도 있습니다. 로컬 캐시를 사용하면 이러한 시나리오에서 지연 시간을 개선하는 데 도움이 됩니다.
- iptables DNAT 및 연결 추적을 건너뛰면 [연결 추적 레이스](https://github.com/kubernetes/kubernetes/issues/56903)를 줄이고 UDP DNS 항목이 연결 추적 테이블을 채우는 것을 방지하는 데 도움이 됩니다.
- 로컬 캐싱 에이전트에서 kube-dns 서비스로의 연결은 TCP로 업그레이드할 수 있습니다. TCP 연결 트랙 항목은 시간 초과를 해야 하는 UDP 항목과 달리 연결 종료 시 제거됩니다(기본값 `nf_conntrack_udp_timeout`은 30초)
- DNS 쿼리를 UDP에서 TCP로 업그레이드하면 삭제된 UDP 패킷과 DNS 타임아웃으로 인한 테일 지연 시간이 보통 최대 30초(3회 재시도 + 10초 타임아웃)까지 줄어듭니다. 노드로컬 캐시가 UDP DNS 쿼리를 듣기 때문에 애플리케이션을 변경할 필요가 없습니다.
- 노드 수준에서 DNS 요청에 대한 메트릭 및 가시성.
- 네거티브 캐싱을 다시 활성화하여 kube-dns 서비스에 대한 쿼리 수를 줄일 수 있습니다.


# NodeLocal DNSCache 설치 - [Docs](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/#configuration)

- NodeLocal DSCache의 로컬 청취 IP 주소는 클러스터의 기존 IP와 충돌하지 않도록 보장할 수 있는 모든 주소일 수 있습니다.
- 예를 들어, IPv4의 '링크-로컬' 범위 '`169.254.0.0/16`' 또는 IPv6 '`fd00::/8`'의 '유니크 로컬 주소' 범위와 같은 로컬 범위의 주소를 사용하는 것이 좋습니다.
- Prepare a manifest similar to the sample [`nodelocaldns.yaml`](https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml) and save it as `nodelocaldns.yaml`.
- **Substitute the variables in the manifest with the right values:**

```bash
kubedns=`kubectl get svc kube-dns -n kube-system -o jsonpath={.spec.clusterIP}` # coredns 의 ClusterIP
domain=<cluster-domain> # 보통 기본값 cluster.local 사용
localdns=<node-local-address> # local listen IP address chosen for NodeLocal DNSCache
```

If kube-proxy is running in IPTABLES mode:
```bash
#
sed -i "s/__PILLAR__LOCAL__DNS__/$localdns/g; s/__PILLAR__DNS__DOMAIN__/$domain/g; s/__PILLAR__DNS__SERVER__/$kubedns/g" nodelocaldns.yaml

- `__PILLAR__CLUSTER__DNS__` and `__PILLAR__UPSTREAM__SERVERS__`는 `node-local-dns` 포드에 의해 채워집니다.
- 이 모드에서는 `node-local-dns` 포드가 **kube-dns 서비스 IP**와 **<node-local-address>**를 모두 수신하므로, 포드는 IP 주소 중 하나를 사용하여 DNS 레코드를 조회할 수 있습니다.
```



If kube-proxy is running in IPVS mode:
```bash
#
sed -i "s/__PILLAR__LOCAL__DNS__/$localdns/g; s/__PILLAR__DNS__DOMAIN__/$domain/g; s/,__PILLAR__DNS__SERVER__//g; s/__PILLAR__CLUSTER__DNS__/$kubedns/g" nodelocaldns.yaml

- 이 모드에서는`node-local-dns` 포드가 **<node-local-address>**에서만 청취합니다.
- IPVS 로드 밸런싱에 사용되는 인터페이스가 이미 이 주소를 사용하고 있기 때문에 `node-local-dns` 인터페이스는 kube-dns 클러스터 IP를 바인딩할 수 없습니다.
- `__PILLAR__UPSTREAM__SERVERS__`는 `node-local-dns` 포드에 의해 채워집니다.
```

- **배포**  `kubectl create -f nodelocaldns.yaml`
- `node-local-dns` 포드가 활성화되면 각 클러스터 노드의 `kube-system` 네임스페이스에서 실행됩니다.
- 이 포드는 캐시 모드에서 CoreDNS를 실행하므로 서로 다른 플러그인이 노출하는 모든 CoreDNS 메트릭을 노드 단위로 사용할 수 있습니다.
- `kubectl delete -f <manifest>`를 사용하여 DaemonSet을 제거하여 비활성화할 수 있습니다. 변경한 내용을 kubetle 설정으로 되돌려야 합니다.
- StubDomains and upstream servers specified in the kube-dns ConfigMap in the kube-system namespace are automatically picked up by node-local-dns pods.


# (참고) Setting memory limits : 캐시 엔트리와 쿼리 처리에 메모리 사용. 10000개 엔트리(30MB 메모리), max_concurrent → VPA - [Docs](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/#setting-memory-limits)

- The `node-local-dns` Pods use **memory** for storing **cache entries** and **processing queries**. Since they do not watch Kubernetes objects, the cluster size or the number of Services / EndpointSlices do not directly affect memory usage. **Memory** **usage** is influenced by the DNS query pattern. From [CoreDNS docs](https://github.com/coredns/deployment/blob/master/kubernetes/Scaling_CoreDNS.md),
    
    <aside> 👉🏻
    
    The default cache size is **10000** **entries**, which uses about **30 MB** when completely filled.
    
    </aside>
    
- This would be the memory usage for each server block (if the cache gets completely filled). Memory usage can be reduced by specifying smaller cache sizes.
    
- The number of concurrent queries is linked to the memory demand, because each extra goroutine used for handling a query requires an amount of memory. You can set an upper limit using the `max_concurrent` option in the forward plugin.
    
- If a `node-local-dns` Pod attempts to use more memory than is available (because of total system resources, or because of a configured [resource limit](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)), the operating system may shut down that pod's container. If this happens, the container that is terminated (“OOMKilled”) does not clean up the custom packet filtering rules that it previously added during startup. The `node-local-dns` container should get restarted (since managed as part of a DaemonSet), but this will lead to a brief DNS downtime each time that the container fails: the packet filtering rules direct DNS queries to a local Pod that is unhealthy.
    
- You can determine a suitable memory limit by running node-local-dns pods without a limit and measuring the peak usage. You can also set up and use a [VerticalPodAutoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) in _recommender mode_, and then check its recommendations.


# NodeLocalDNS 실습 + Cilium Local Redirect Policy

^bd1807

NodeLocal DNSCache 설치 및 확인 - [Docs](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/#configuration) , [Github](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns/nodelocaldns)
```bash
# iptables 확인
iptables-save | tee before.txt

#
wget https://github.com/kubernetes/kubernetes/raw/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml

# kubedns 는 coredns 서비스의 ClusterIP를 변수 지정
kubedns=`kubectl get svc kube-dns -n kube-system -o jsonpath={.spec.clusterIP}`
domain='cluster.local'    ## default 값
localdns='169.254.20.10'  ## default 값
echo $kubedns $domain $localdns

# iptables 모드 사용 중으로 아래 명령어 수행
sed -i "s/__PILLAR__LOCAL__DNS__/$localdns/g; s/__PILLAR__DNS__DOMAIN__/$domain/g; s/__PILLAR__DNS__SERVER__/$kubedns/g" nodelocaldns.yaml

# nodelocaldns 설치
kubectl apply -f nodelocaldns.yaml

#
kubectl get pod -n kube-system -l k8s-app=node-local-dns -owide
NAME                   READY   STATUS    RESTARTS   AGE   IP               NODE      NOMINATED NODE   READINESS GATES
node-local-dns-9xrv9   1/1     Running   0          41s   192.168.10.100   k8s-ctr   <none>           <none>

#
kubectl edit cm -n kube-system node-local-dns # 'cluster.local' 과 '.:53' 에 log, debug 추가
kubectl -n kube-system rollout restart ds node-local-dns

kubectl describe cm -n kube-system node-local-dns
...
cluster.local:53 {
    errors
    cache {
            success 9984 30
            denial 9984 5
    }
    reload
    loop
    bind 169.254.20.10 10.96.0.10
    forward . __PILLAR__CLUSTER__DNS__ {
            force_tcp
    }
    prometheus :9253
    health 169.254.20.10:8080
    }
    ...
.:53 {
    errors
    cache 30
    reload
    loop
    bind 169.254.20.10 10.96.0.10
    forward . __PILLAR__UPSTREAM__SERVERS__
    prometheus :9253
    }
...

# iptables 확인 : 규칙 업데이트까지 다소 시간 소요!
iptables-save | tee after.txt
diff before.txt after.txt

##
iptables -t filter -S | grep -i dns
-A INPUT -d 10.96.0.10/32 -p udp -m udp --dport 53 -m comment --comment "NodeLocal DNS Cache: allow DNS traffic" -j ACCEPT
-A INPUT -d 10.96.0.10/32 -p tcp -m tcp --dport 53 -m comment --comment "NodeLocal DNS Cache: allow DNS traffic" -j ACCEPT
-A INPUT -d 169.254.20.10/32 -p udp -m udp --dport 53 -m comment --comment "NodeLocal DNS Cache: allow DNS traffic" -j ACCEPT
-A INPUT -d 169.254.20.10/32 -p tcp -m tcp --dport 53 -m comment --comment "NodeLocal DNS Cache: allow DNS traffic" -j ACCEPT
-A OUTPUT -s 10.96.0.10/32 -p udp -m udp --sport 53 -m comment --comment "NodeLocal DNS Cache: allow DNS traffic" -j ACCEPT
-A OUTPUT -s 10.96.0.10/32 -p tcp -m tcp --sport 53 -m comment --comment "NodeLocal DNS Cache: allow DNS traffic" -j ACCEPT
-A OUTPUT -s 169.254.20.10/32 -p udp -m udp --sport 53 -m comment --comment "NodeLocal DNS Cache: allow DNS traffic" -j ACCEPT
-A OUTPUT -s 169.254.20.10/32 -p tcp -m tcp --sport 53 -m comment --comment "NodeLocal DNS Cache: allow DNS traffic" -j ACCEPT

##
iptables -t raw -S | grep -i dns
-A PREROUTING -d 10.96.0.10/32 -p udp -m udp --dport 53 -m comment --comment "NodeLocal DNS Cache: skip conntrack" -j NOTRACK
-A PREROUTING -d 10.96.0.10/32 -p tcp -m tcp --dport 53 -m comment --comment "NodeLocal DNS Cache: skip conntrack" -j NOTRACK
-A PREROUTING -d 169.254.20.10/32 -p udp -m udp --dport 53 -m comment --comment "NodeLocal DNS Cache: skip conntrack" -j NOTRACK
-A PREROUTING -d 169.254.20.10/32 -p tcp -m tcp --dport 53 -m comment --comment "NodeLocal DNS Cache: skip conntrack" -j NOTRACK
-A OUTPUT -s 10.96.0.10/32 -p tcp -m tcp --sport 8080 -m comment --comment "NodeLocal DNS Cache: skip conntrack" -j NOTRACK
-A OUTPUT -d 10.96.0.10/32 -p tcp -m tcp --dport 8080 -m comment --comment "NodeLocal DNS Cache: skip conntrack" -j NOTRACK
...

# logs : 
kubectl -n kube-system logs -l k8s-app=kube-dns -f
kubectl -n kube-system logs -l k8s-app=node-local-dns -f

#
kubectl exec -it curl-pod -- cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5

# 
kubectl exec -it curl-pod -- nslookup webpod
kubectl exec -it curl-pod -- nslookup google.com

#
kubectl delete pod curl-pod

cat << EOF | kubectl apply -f -
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

kubectl exec -it curl-pod -- cat /etc/resolv.conf

# 로그 확인 시 현재 nodelocaldns 미활용! 
kubectl -n kube-system logs -l k8s-app=kube-dns -f
kubectl -n kube-system logs -l k8s-app=node-local-dns -f

#
kubectl exec -it curl-pod -- nslookup webpod
kubectl exec -it curl-pod -- nslookup google.com

```


# Cilium Local Redirect Policy : --set localRedirectPolicy=true - [Docs](https://docs.cilium.io/en/stable/network/kubernetes/local-redirect-policy/)

- IP 주소와 Port/Protocol tuple 또는 **Kubernetes Service** 로 향하는 포드 **트래픽**을 eBPF를 사용하여 노드 내 **백엔드 포드로 로컬로 리디렉션**할 수 있도록 하는 Cilium의 로컬 리디렉션 정책을 구성하는 방법을 설명합니다.
- 백엔드 포드의 네임스페이스는 정책의 네임스페이스와 일치해야 합니다.
- CiliumLocalRedirectPolicy는 CustomResourceDefinition으로 구성되어 있습니다.
node-local-dns configmap 내용
```bash 
Corefile:
----
cluster.local:53 {
    log
    debug
    errors
    cache {
            success 9984 30
            denial 9984 5
    }
    reload
    loop
    bind 0.0.0.0
    forward . __PILLAR__CLUSTER__DNS__ {
            force_tcp
    }
    prometheus :9253
    health
    }
in-addr.arpa:53 {
    errors
    cache 30
    reload
    loop
    bind 0.0.0.0
    forward . __PILLAR__CLUSTER__DNS__ {
            force_tcp
    }
    prometheus :9253
    }
ip6.arpa:53 {
    errors
    cache 30
    reload
    loop
    bind 0.0.0.0
    forward . __PILLAR__CLUSTER__DNS__ {
            force_tcp
    }
    prometheus :9253
    }
.:53 {
    log
    debug
    errors
    cache 30
    reload
    loop
    bind 0.0.0.0
    forward . __PILLAR__UPSTREAM__SERVERS__
    prometheus :9253
    }
```

```bash
#
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
  --set localRedirectPolicy=true

kubectl rollout restart deploy cilium-operator -n kube-system
kubectl rollout restart ds cilium -n kube-system

#
wget https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/kubernetes-local-redirect/node-local-dns.yaml

kubedns=$(kubectl get svc kube-dns -n kube-system -o jsonpath={.spec.clusterIP})
sed -i "s/__PILLAR__DNS__SERVER__/$kubedns/g;" node-local-dns.yaml
vi -d nodelocaldns.yaml node-local-dns.yaml

## before
args: [ "-localip", "169.254.20.10,10.96.0.10", "-conf", "/etc/Corefile", "-upstreamsvc", "kube-dns-upstream" ]

## after
args: [ "-localip", "169.254.20.10,10.96.0.10", "-conf", "/etc/Corefile", "-upstreamsvc", "kube-dns-upstream", "-skipteardown=true", "-setupinterface=false", "-setupiptables=false" ]


# 배포
# Modify Node-local DNS cache’s deployment yaml to pass these additional arguments to node-cache: 
## -skipteardown=true, -setupinterface=false, and -setupiptables=false.

# Modify Node-local DNS cache’s deployment yaml to put it in non-host namespace by setting hostNetwork: false for the daemonset.
# In the Corefile, bind to 0.0.0.0 instead of the static IP.
kubectl apply -f node-local-dns.yaml

#
kubectl edit cm -n kube-system node-local-dns # log, debug 추가
kubectl -n kube-system rollout restart ds node-local-dns

kubectl describe cm -n kube-system node-local-dns
cluster.local:53 {
    errors
    cache {
            success 9984 30
            denial 9984 5
    }
    reload
    loop
    bind 0.0.0.0
    forward . __PILLAR__CLUSTER__DNS__ {
            force_tcp
    }
    prometheus :9253
    health
    }
    ...
.:53 {
    errors
    cache 30
    reload
    loop
    bind 0.0.0.0
    forward . __PILLAR__UPSTREAM__SERVERS__
    prometheus :9253
    }


#
wget https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/kubernetes-local-redirect/node-local-dns-lrp.yaml
cat node-local-dns-lrp.yaml | yq
apiVersion: "cilium.io/v2"
kind: CiliumLocalRedirectPolicy
metadata:
  name: "nodelocaldns"
  namespace: kube-system
spec:
  redirectFrontend:
    serviceMatcher:
      serviceName: kube-dns
      namespace: kube-system
  redirectBackend:
    localEndpointSelector:
      matchLabels:
        k8s-app: node-local-dns
    toPorts:
      - port: "53"
        name: dns
        protocol: UDP
      - port: "53"
        name: dns-tcp
        protocol: TCP
        
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/kubernetes-local-redirect/node-local-dns-lrp.yaml

#
kubectl get CiliumLocalRedirectPolicy -A
NAMESPACE     NAME           AGE
kube-system   nodelocaldns   5m26s

#
kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium-dbg lrp list
LRP namespace   LRP name       FrontendType                Matching Service
kube-system     nodelocaldns   clusterIP + all svc ports   kube-system/kube-dns
                |              10.96.0.10:53/UDP -> 172.20.0.52:53(kube-system/node-local-dns-mhv4p), 
                |              10.96.0.10:53/TCP -> 172.20.0.52:53(kube-system/node-local-dns-mhv4p), 
                |              10.96.0.10:9153/TCP -> 

kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium-dbg service list | grep LocalRedirect
2    10.96.0.10:53/UDP     LocalRedirect   1 => 172.20.0.52:53/UDP (active)        
3    10.96.0.10:53/TCP     LocalRedirect   1 => 172.20.0.52:53/TCP (active)


# logs
kubectl -n kube-system logs -l k8s-app=kube-dns -f
kubectl -n kube-system logs -l k8s-app=node-local-dns -f

#
kubectl exec -it curl-pod -- nslookup www.google.com


# nodelocaldns 에 캐시된 정보로 바로 질의 응답 확인!
kubectl -n kube-system logs -l k8s-app=node-local-dns -f
[INFO] 172.20.0.119:59966 - 48193 "A IN www.google.com.default.svc.cluster.local. udp 58 false 512" NXDOMAIN qr,aa,rd 151 0.057756155s
[INFO] 172.20.0.119:57117 - 36265 "A IN www.google.com.svc.cluster.local. udp 50 false 512" NXDOMAIN qr,aa,rd 143 0.000990079s
[INFO] 172.20.0.119:57568 - 25713 "A IN www.google.com.cluster.local. udp 46 false 512" NXDOMAIN qr,aa,rd 139 0.000896198s
[INFO] 172.20.0.119:58204 - 42574 "A IN www.google.com. udp 32 false 512" NOERROR qr,rd,ra 62 0.035130807s
[INFO] 172.20.0.119:46673 - 7830 "AAAA IN www.google.com. udp 32 false 512" NOERROR qr,rd,ra 74 0.007996413s


```


## (참고) cilium 재설치 : --set cleanBpfState=true --set cleanState=true
```bash
#
helm uninstall -n kube-system cilium

#
iptables-save | grep -v KUBE | grep -v CILIUM | iptables-restore
iptables-save

sshpass -p 'vagrant' ssh vagrant@k8s-w1 "sudo iptables-save | grep -v KUBE | grep -v CILIUM | sudo iptables-restore"
sshpass -p 'vagrant' ssh vagrant@k8s-w1 sudo iptables-save

#
helm install cilium cilium/cilium --version 1.17.6 --namespace kube-system \
--set k8sServiceHost=192.168.10.100 --set k8sServicePort=6443 \
--set ipam.mode="cluster-pool" --set ipam.operator.clusterPoolIPv4PodCIDRList={"172.20.0.0/16"} --set ipv4NativeRoutingCIDR=172.20.0.0/16 \
--set routingMode=native --set autoDirectNodeRoutes=true --set endpointRoutes.enabled=true \
--set kubeProxyReplacement=true --set bpf.masquerade=true --set installNoConntrackIptablesRules=true \
--set endpointHealthChecking.enabled=false --set healthChecking=false \
--set hubble.enabled=true --set hubble.relay.enabled=true --set hubble.ui.enabled=true \
--set hubble.ui.service.type=NodePort --set hubble.ui.service.nodePort=30003 \
--set prometheus.enabled=true --set operator.prometheus.enabled=true --set hubble.metrics.enableOpenMetrics=true \
--set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}" \
--set operator.replicas=1 --set debug.enabled=true --set cleanBpfState=true --set cleanState=true

#
k9s → pod → 0 (all) 
```