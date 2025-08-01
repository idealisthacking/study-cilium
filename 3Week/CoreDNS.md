CoreDNS 소개 - [Docs](https://kubernetes.io/ko/docs/tasks/administer-cluster/dns-custom-nameservers/) , [Home](https://coredns.io/manual/toc/) , [Plugins](https://coredns.io/plugins/) , [Youtube](https://www.youtube.com/watch?v=W3f5Ks0j2Q8)

```bash
# 파드의 DNS 설정 정보 확인
kubectl exec -it curl-pod -- cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5

#
cat /var/lib/kubelet/config.yaml | grep cluster -A1
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local

#
kubectl get svc,ep -n kube-system kube-dns
NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   173m

NAME                 ENDPOINTS                                                     AGE
endpoints/kube-dns   172.20.0.199:53,172.20.1.162:53,172.20.0.199:53 + 3 more...   172m

kubectl get pod -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS      AGE
coredns-674b8bbfcf-2x9gs   1/1     Running   0             162m
coredns-674b8bbfcf-w5d8m   1/1     Running   2 (12m ago)   162m

#
kc describe pod -n kube-system -l k8s-app=kube-dns
...
  config-volume:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      coredns
    Optional:  false
...

kc describe cm -n kube-system coredns
...
Corefile:
----
.:53 {              # 모든 도메인 요청을 53포트에서 수신
    errors          # DNS 응답 중 에러가 발생할 경우 로그 출력
    health {        # health 엔드포인트를 제공하여 상태 확인 가능
       lameduck 5s  # 종료 시 5초간 lameduck 모드로 트래픽을 점차 줄이며 종료
    }
    ready           # ready 엔드포인트 제공, 8181 포트의 HTTP 엔드포인트가, 모든 플러그인이 준비되었다는 신호를 보내면 200 OK 를 반환
    kubernetes cluster.local in-addr.arpa ip6.arpa {    # Kubernetes DNS 플러그인 설정(클러스터 내부 도메인 처리), cluster.local: 클러스터 도메인
       pods insecure                         # 파드 IP로 DNS 조회 허용 (보안 없음)
       fallthrough in-addr.arpa ip6.arpa     #  해당 도메인에서 결과 없으면 다음 플러그인으로 전달
       ttl 30                                #  캐시 타임 (30초)
    }
    prometheus :9153 # Prometheus metrics 수집 가능
    forward . /etc/resolv.conf {             # CoreDNS가 모르는 도메인은 지정된 업스트림(보통 외부 DNS)으로 전달, .: 모든 쿼리
       max_concurrent 1000                   # 병렬 포워딩 최대 1000개
    }
    cache 30 {                        # DNS 응답 캐시 기능, 기본 캐시 TTL 30초
       disable success cluster.local  # 성공 응답 캐시 안 함 (cluster.local 도메인)
       disable denial cluster.local   # NXDOMAIN 응답도 캐시 안 함
    } 
    loop         # 간단한 전달 루프(loop)를 감지하고, 루프가 발견되면 CoreDNS 프로세스를 중단(halt).
    reload       # Corefile 이 변경되었을 때 자동으로 재적용, 컨피그맵 설정을 변경한 후에 변경 사항이 적용되기 위하여 약 2분정도 소요.
    loadbalance  # 응답에 대하여 A, AAAA, MX 레코드의 순서를 무작위로 선정하는 라운드-로빈 DNS 로드밸런서.
}

#
cat /etc/resolv.conf
nameserver 127.0.0.53
options edns0 trust-ad
search .

resolvectl 
Link 2 (eth0)
    Current Scopes: DNS
         Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 10.0.2.3
       DNS Servers: 10.0.2.3
```

## (참고) forward 플러그인 - [Docs](https://coredns.io/plugins/forward/)
```bash
# 활용 1 : '.consul.local' 도메인을 관리하는 도메인 서버가 존재 시, coredns 에서 해당 도메인 서버로 질의 설정 시
consul.local:53 {
    errors
    cache 30
    forward . 10.150.0.1
}

# 활용 2 : 모든 비 클러스터의 DNS 조회가 172.16.0.1 의 특정 네임서버 사용 시, /etc/resolv.conf 대신 forward 를 네임서버로 지정
forward .  172.16.0.1


# 위 1,2 포함한 설정 예시
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 172.16.0.1  # 활용 2
        cache 30
        loop
        reload
        loadbalance
    }
    consul.local:53 {         # 활용 1
        errors
        cache 30
        forward . 10.150.0.1
    }  
```

## 파드에서 DNS 질의 확인 - [Docs](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/) , [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) , [Autoscale the DNS Service in a Cluster](https://kubernetes.io/docs/tasks/administer-cluster/dns-horizontal-autoscaling/)

```bash
# 모니터링1
cilium hubble port-forward&
hubble observe -f --port 53
hubble observe -f --port 53 --protocol UDP

# 모니터링2
tcpdump -i any udp port 53 -nn

# 파드 IP 확인
kubectl get pod -owide
NAME                      READY   STATUS    RESTARTS      AGE     IP             NODE      NOMINATED NODE   READINESS GATES
curl-pod                  1/1     Running   2 (51m ago)   3h21m   172.20.1.59    k8s-ctr   <none>           <none>

kubectl get pod -n kube-system -l k8s-app=kube-dns -owide
NAME                       READY   STATUS    RESTARTS      AGE     IP             NODE      NOMINATED NODE   READINESS GATES
coredns-674b8bbfcf-2x9gs   1/1     Running   0             3h23m   172.20.0.199   k8s-w1    <none>           <none>
coredns-674b8bbfcf-w5d8m   1/1     Running   2 (53m ago)   3h23m   172.20.1.162   k8s-ctr   <none>           <none>

kubectl exec -it curl-pod -- cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local


# 실습 편리를 위해 coredns 파드를 1개로 축소
kubectl scale deployment -n kube-system coredns --replicas 1
kubectl get pod -n kube-system -l k8s-app=kube-dns -owide

#
kubectl exec -it curl-pod -- curl kube-dns.kube-system.svc:9153/metrics | grep coredns_cache_ | grep -v ^#


# 도메인 질의
kubectl exec -it curl-pod -- nslookup webpod
kubectl exec -it curl-pod -- nslookup -debug webpod

kubectl exec -it curl-pod -- nslookup -debug google.com


# coredns 로깅, 디버깅 활성화
k9s → configmap → coredns 선택 → E(edit) → 아래처럼 log, debug 입력 후 빠져나오기
    .:53 {
        log
        debug
        errors

# 로그 모니터링 3
kubectl -n kube-system logs -l k8s-app=kube-dns -f


# 도메인 질의
kubectl exec -it curl-pod -- nslookup webpod
kubectl exec -it curl-pod -- nslookup google.com


# CoreDNS가 prometheus 플러그인을 사용하고 있다면, 메트릭 포트(:9153)를 통해 캐시 관련 정보를 수집.
## coredns_cache_entries 현재 캐시에 저장된 엔트리(항목) 수 : type: success 또는 denial (정상 응답 or NXDOMAIN 등)
## coredns_cache_hits_total	캐시 조회 성공 횟수
## coredns_cache_misses_total	캐시 미스 횟수
## coredns_cache_requests_total	캐시 관련 요청 횟수의 총합

kubectl exec -it curl-pod -- curl kube-dns.kube-system.svc:9153/metrics | grep coredns_cache_ | grep -v ^#
coredns_cache_entries{server="dns://:53",type="denial",view="",zones="."} 1
coredns_cache_entries{server="dns://:53",type="success",view="",zones="."} 0
coredns_cache_hits_total{server="dns://:53",type="success",view="",zones="."} 12
coredns_cache_misses_total{server="dns://:53",view="",zones="."} 517
coredns_cache_requests_total{server="dns://:53",view="",zones="."} 529

```