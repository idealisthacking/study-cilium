![[Pasted image 20250803210430.png]]

- **LB IPAM**은 **Cilium**이 IP 주소를 **LoadBalancer** 유형의 **서비스**에 할**당**할 수 있게 해주는 기능입니다. 이 기능은 일반적으로 클라우드 제공업체에 맡겨지지만, 프라이빗 클라우드 환경에 배포할 때 이러한 기능을 항상 사용할 수 있는 것은 아닙니다.
- LB IPAM은 `Cilium BGP Control Plane` 및 `L2 Announcements` / `L2 Aware LB`(베타)와 같은 기능과 함께 작동합니다. LB IPAM이 서비스 객체 및 기타 기능에 IP를 할당하고 할당하는 역할을 하는 곳은 이러한 IP의 **로드 밸런싱** 및/또는 **광고**를 담당합니다.
- Cilium BGP Control Plane을 사용하여 LB IPAM이 할당한 IP 주소를 `BGP를 통해 광고`하고 L2 Announcements / L2 Aware LB(베타)를 통해 `로컬로 광고`합니다.
- LB IPAM은 항상 활성화되어 있지만 휴면 상태입니다. 첫 번째 IP 풀이 클러스터에 추가되면 컨트롤러가 활성화됩니다.
- 기능
    - Service Selectors - [Docs](https://docs.cilium.io/en/stable/network/lb-ipam/#service-selectors)
    - Disabling a Pool - [Docs](https://docs.cilium.io/en/stable/network/lb-ipam/#disabling-a-pool)
    - Service 사용 확인 - [Docs](https://docs.cilium.io/en/stable/network/lb-ipam/#services)
    - LoadBalancerClass : BGP or L2 지정 - [Docs](https://docs.cilium.io/en/stable/network/lb-ipam/#loadbalancerclass)
    - Requesting IPs : 특정 Service에 EX-IP를 직접 설정 - [Docs](https://docs.cilium.io/en/stable/network/lb-ipam/#loadbalancerclass)
    - Sharing Keys : EX-IP 1개를 각기 다른 Port 를 통해서 사용 - [Docs](https://docs.cilium.io/en/stable/network/lb-ipam/#requesting-ips)


# [k8s 클러스터 내부] webpod 서비스를 LoadBalancer Type 설정 with Cilium LB IPAM

```bash
# cilium ip pool 생성
kubectl get CiliumLoadBalancerIPPool -A

# 충돌나지 않는지 대역 확인 할 것!
cat << EOF | kubectl apply -f -
apiVersion: "cilium.io/v2"  # v1.17 : cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: "cilium-lb-ippool"
spec:
  blocks:
  - start: "192.168.10.211"
    stop:  "192.168.10.215"
EOF
```

```bash
# CiliumLoadBalancerIPPool 축약어 : ippools,ippool,lbippool,lbippools
kubectl api-resources | grep -i CiliumLoadBalancerIPPool
ciliumloadbalancerippools           ippools,ippool,lbippool,lbippools   cilium.io/v2               false        CiliumLoadBalancerIPPool

kubectl get ippools
NAME               DISABLED   CONFLICTING   IPS AVAILABLE   AGE
cilium-lb-ippool   false      False         5               20s


# webpod 서비스를 LoadBalancer Type 변경 설정
kubectl patch svc webpod -p '{"spec":{"type":"LoadBalancer"}}'

# 확인
kubectl get svc webpod
NAME     TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
webpod   LoadBalancer   10.96.32.212   192.168.10.211   80:32039/TCP   150m


# LBIP로 curl 요청 확인 : k8s 노드들에서 LB EXIP로 통신 가능!
kubectl get svc webpod -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
LBIP=$(kubectl get svc webpod -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

curl -s $LBIP
kubectl exec -it curl-pod -- curl -s $LBIP
kubectl exec -it curl-pod -- curl -s $LBIP | grep Hostname   # 대상 파드 이름 출력
kubectl exec -it curl-pod -- curl -s $LBIP | grep RemoteAddr # 대상 파드 입장에서 소스 IP 출력(Layer3)

# 반복 접속 : 
while true; do kubectl exec -it curl-pod -- curl -s $LBIP | grep Hostname; sleep 0.1; done
for i in {1..100};  do kubectl exec -it curl-pod -- curl -s $LBIP | grep Hostname; done | sort | uniq -c | sort -nr


# IP 할당 확인
kubectl get ippools
kubectl get ippools -o jsonpath='{.items[*].status.conditions[?(@.type!="cilium.io/PoolConflict")]}' | jq

kubectl get svc webpod -o json | jq
kubectl get svc webpod -o jsonpath='{.status}' | jq

```

# [k8s 클러스터 외부] webpod 서비스를 LoadBalancer External IP로 호출 확인 → 도움 : BGP 혹은 L2 Announcements / L2 Aware LB(Beta)

Router -> Node IP

```bash
# router : K8S 외부에서 통신 불가! 
LBIP=192.168.10.211
curl --connect-timeout 1 $LBIP
arping -i eth1 $LBIP -c 1
arping -i eth1 $LBIP -c 100000
...
```

