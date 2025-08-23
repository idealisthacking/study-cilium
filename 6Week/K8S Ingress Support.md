# Cilium K8S Ingress Support 소개 - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/ingress/)

- Cilium uses the standard [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) resource definition, with an `ingressClassName` of `cilium`.
- This can be used for path-based routing and for TLS termination. For backwards compatibility, he `kubernetes.io/ingress.class` annotation with value of `cilium` is also supported.
- The ingress controller creates a Service of LoadBalancer type, so your environment will need to support this.
- Cilium allows you to specify load balancer mode for the Ingress resource:
    - `dedicated`: The Ingress controller will create a dedicated loadbalancer for the Ingress.
    - `shared`: The Ingress controller will use a shared loadbalancer for all Ingress resources.
- Each load balancer mode has its own benefits and drawbacks. The shared mode saves resources by sharing a single LoadBalancer config across all Ingress resources in the cluster, while the dedicated mode can help to avoid potential conflicts (e.g. path prefix) between resources.
- It is possible to change the load balancer mode for an Ingress resource.
- When the mode is changed, active connections to backends of the Ingress may be terminated during the reconfiguration due to a new load balancer IP address being assigned to the Ingress resource. _→ 기존 모드 변경 시 LB IP 변경되어 Ingress 백엔드 활성 연결 종료됨._

- 필수 조건
    - Cilium must be configured with NodePort enabled, using `nodePort.enabled=true` or by enabling the kube-proxy replacement with `kubeProxyReplacement=true`. For more information, see [kube-proxy replacement](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#kubeproxy-free).
    - Cilium must be configured with the L7 proxy enabled using `l7Proxy=true` (enabled by default).
    - By default, the Ingress controller creates a Service of LoadBalancer type, so your environment will need to support this. Alternatively, you can change this to NodePort or, since Cilium 1.16+, directly expose the Cilium L7 proxy on the [host network](https://docs.cilium.io/en/stable/network/servicemesh/ingress/#gs-ingress-host-network-mode).

- How Cilium Ingress and Gateway API differ from other Ingress controllers
    - Cilium의 Ingress 및 Gateway API 지원과 다른 Ingress 컨트롤러 간의 가장 큰 차이점 중 하나는 구현이 CNI와 얼마나 밀접하게 연결되어 있는지입니다. Cilium의 경우 Ingress와 Gateway API는 네트워킹 스택의 일부이므로 다른 Ingress 또는 Gateway API 컨트롤러(심지어 Cilium 클러스터에서 실행되는 다른 Ingress 또는 Gateway API 컨트롤러)와는 다른 방식으로 작동합니다.
        
    - 다른 Ingress 또는 Gateway API 컨트롤러는 일반적으로 클러스터에 Deployment 또는 데몬셋으로 설치되며, Loadbalancer Service 또는 이와 유사한 서비스를 통해 노출됩니다. (물론 Cilium은 이를 활성화할 수 있습니다).
        
    - Cilium Ingress 및 Gateway API 구성은 Loadbalancer 또는 NodePort 서비스를 통해 노출되거나, 선택적으로 Host network에서도 노출될 수 있습니다. 이 모든 경우에 트래픽이 서비스의 포트에 도착하면 eBPF 코드가 트래픽을 가로채어 투명하게 Envoy에 전달합니다(TPROXY 커널 기능을 사용하여).
        
        - 아래는 ChatGPT 내용으로 부정확할 수 있음
            - _패킷이 Kubernetes Service IP:Port에 도착하면, kube-proxy 없이 Cilium의 eBPF가 이를 가로챔._
            - _Cilium은 Envoy Proxy를 L7 데이터플레인으로 사용. 하지만 단순 포트포워딩(REDIRECT)이 아니라, 커널의 TPROXY(Transparent Proxy) 기능을 이용함._
                - _커널 netfilter/iptables에서 지원하는 특수 기능_
                - _‘투명 프록시’ 역할 : 클라이언트는 원래 목적지로 연결했다고 생각하지만, 사실상 로컬 프록시(여기서는 Envoy)로 트래픽이 리디렉션됨_
                - _원본 destination IP/Port, source IP/Port를 유지한 채 소켓 레벨에서 Envoy가 트래픽을 가로채어 처리할 수 있음_
                - _즉, Envoy가 "내가 80/443 포트에서 직접 리스닝한 것처럼" 패킷을 받을 수 있고, 그 과정은 클라이언트 입장에서 완전히 투명합니다._
            - 동작 경로
                - [Client] → [K8s Node:Ingress/Gateway Service Port]
                    
                    → (eBPF Service LB) → (TPROXY) → [Envoy Proxy (Pod)]
                    
                    → (L7 라우팅/정책 처리) → (eBPF) → [Backend Pod]
                    
    - 이는 Client IP visibility 같은 문제에 영향을 미치며, 이는 다른 입력 컨트롤러에 대한 Cilium의 Ingress 또는 Gateway API과 다르게 작동합니다.
        
    - 또한 Cilium의 네트워크 정책 엔진이 Ingress에서 들어오는 트래픽 경계와 트래픽에 CiliumNetworkPolicy를 적용할 수 있도록 합니다.

- Cilium’s ingress config and CiliumNetworkPolicy
    - 노드별 envoy proxy 에 Network policy 정책 적용할 수 있다.
    - Ingress and Gateway API traffic bound to backend services via Cilium passes through a per-node Envoy proxy.
    - The per-node Envoy proxy has special code that allows it to interact with the eBPF policy engine, and do policy lookups on traffic.
    - This allows Envoy to be a Network Policy enforcement point, both for Ingress (and Gateway API) traffic, and also for east-west traffic via GAMMA or L7 Traffic Management.
    - However, for ingress config, there’s also an additional step. Traffic that arrives at Envoy for Ingress or Gateway API is assigned the special ingress identity in Cilium’s Policy engine. _Cilium의 정책 엔진에서 특수 입력 ID가 할당._
    - 클러스터 외부에서 들어오는 트래픽은 일반적으로 클러스터에 (IP CIDR 정책이 없는 한) world identity가 할당됩니다.
    - 이는 실제로 실리움 인그레스에 두 개의 논리적 정책 집행 지점이 있다는 것을 의미합니다. two logical Policy enforcement points.
    - before traffic arrives at the ingress identity, and after, when it is about to exit the per-node Envoy.
![[Pasted image 20250817211615.png]]

* This means that, when applying Network Policy to a cluster, it’s important to ensure that both steps are allowed, and that traffic is allowed from world to ingress, and from ingress to identities in the cluster (like the productpage identity in the image above). although the same principles also apply for Gateway API. Gateway API도 동일하게 적용된다.


- Source IP Visibility
    - By default, Cilium’s Envoy instances are configured to append the visible source address of incoming HTTP connections to the `X-Forwarded-For` header, using the usual rules.
    - That is, by default Cilium sets the number of `trusted hops` to `0`, indicating that Envoy should use the address the connection is opened from, rather than a value inside the `X-Forwarded-For list`.
    - `Increasing` this count will have Envoy use the `n` th value from the list, counting from the `right`.
    - Envoy will also set the X-Envoy-External-Address header to the trusted client address, whatever that turns out to be, based on X-Forwarded-For.


- TLS Passthrough and source IP visibility
    - Ingress와 Gateway API는 모두 TLS Passthrough 구성을 지원합니다(Ingress는 annotation, Gateway API는 TLSRoute 리소스를 통해).
    - 이 구성을 통해 여러 TLS Passthrough 백엔드가 로드밸런서에서 동일한 TLS 포트를 공유할 수 있으며, Envoy은 TLS 핸드셰이크의 서버 이름 표시기(SNI) 필드를 검사하고 이를 사용하여 TLS 스트림을 백엔드로 전달합니다.
    - 그러나 이는 소스 IP visibility 문제를 제기합니다. 왜냐하면 Envoy이 TLS 스트림의 TCP 프록시를 수행하고 있기 때문입니다.
    - TLS 트래픽이 Envoy에 도착하여 TCP 스트림을 종료하고, Envoy이 클라이언트 hello를 검사하여 SNI를 찾고, 전달할 백엔드를 선택한 다음, 새로운 TCP 스트림을 시작하고, 다운스트림(외부) 패킷 내부의 TLS 트래픽을 업스트림(백엔드)으로 전달하는 방식입니다.
    - 새로운 TCP 스트림이기 때문에 백엔드에 관한 한 소스 IP는 Envoy(실리움 구성에 따라 노드 IP인 경우가 많습니다)입니다.
    - TLS Passthrough 수행할 때 백엔드는 전달된 TLS 스트림의 소스로 Cilium Envoy IP 주소를 인식합니다.

- Ingress Path Types and Precedence
    - The Ingress specification supports three types of paths:
        - 정확히 Exact - match the given path exactly.
        - 접두사 Prefix - match the URL path prefix split by `/`.
            - The last path segment must match the whole segment -
            - if you configure a Prefix path of `/foo/bar`, `/foo/bar/baz` will match, but `/foo/barbaz` will not.
        - IngressClass에 맡김 ImplementationSpecific
            - Interpretation of the Path is up to the IngressClass. In Cilium’s case, we define ImplementationSpecific to be “Regex”, so Cilium will interpret any given path as a regular expression and program Envoy accordingly.
            - Notably, some other implementations have ImplementationSpecific mean “Prefix”, and in those cases, Cilium will treat the paths differently.
            - (Since a path like `/foo/bar` contains no regex characters, when it is configured in Envoy as a regex, it will function as an `Exact` match instead).

- When multiple path types are configured on an Ingress object, Cilium will configure Envoy with the matches in the following order:
    1. Exact
    2. ImplementationSpecific (that is, regular expression)
    3. Prefix
    4. The `/` Prefix match has special handling and always goes last.

- Within each of these path types, the paths are sorted in decreasing order of string length.
- If you do use ImplementationSpecific regex support, be careful with using the `*` operator, since it will increase the length of the regex, but may match another, shorter option.
- For example, if you have two ImplementationSpecific paths, `/impl`, and `/impl.*`, the second will be sorted ahead of the first in the generated config. But because `*` is in use, the `/impl` match will never be hit, as any request to that path will match the `/impl.*` path first.
- See the [Ingress Path Types](https://docs.cilium.io/en/stable/network/servicemesh/path-types/#gs-ingress-path-types) for more information.

- Supported Ingress Annotations : 지원되는 인그레스 애너테이션 확인 - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/ingress/#supported-ingress-annotations)
- Host network mode : expose the Cilium ingress controller (Envoy listener) directly on the host network
    - This is useful in cases where a LoadBalancer Service is unavailable, such as in development environments or environments with cluster-external loadbalancers.
    - Bind to privileged port
- Deploy Cilium Ingress listeners on subset of nodes : 특정 노드만 노출 - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/ingress/#deploy-cilium-ingress-listeners-on-subset-of-nodes)


eBPF Datapath : Ingress to → Endpoint - [Docs](https://docs.cilium.io/en/stable/network/ebpf/) , [Youtube](https://www.youtube.com/watch?v=0BKU6avwS98)
![[Pasted image 20250817211907.png]]


# Cilium K8S Ingress Support 관련 정보 확인 : Cilium Ingress 와 Cilium GatewayAPI는 동시 활성화 불가능함. 다른 Ingress(nginx) 와는 가능.

```bash
# cilium 설치 시 아래 파라미터 적용되어 있음
## --set ingressController.enabled=true
## --set ingressController.loadbalancerMode=shared
## --set loadBalancer.l7.backend=envoy \
cilium config view | grep -E '^loadbalancer|l7'
enable-l7-proxy                                   true
loadbalancer-l7                                   envoy
loadbalancer-l7-algorithm                         round_robin
loadbalancer-l7-ports 

# ingress 에 예약된 내부 IP 확인 : node(cilium-envoy) 별로 존재
kubectl exec -it -n kube-system ds/cilium -- cilium ip list | grep ingress
172.20.0.248/32     reserved:ingress                                                                    
172.20.1.35/32      reserved:ingress

# cilium-envoy 확인
kubectl get ds -n kube-system cilium-envoy -owide
kubectl get pod -n kube-system -l k8s-app=cilium-envoy -owide
NAME                 READY   STATUS    RESTARTS   AGE   IP               NODE      NOMINATED NODE   READINESS GATES
cilium-envoy-6hc4t   1/1     Running   0          92m   192.168.10.100   k8s-ctr   <none>           <none>
cilium-envoy-cssvq   1/1     Running   0          90m   192.168.10.101   k8s-w1    <none>           <none>

kc describe pod -n kube-system -l k8s-app=cilium-envoy
...
Containers:
  cilium-envoy:
    Container ID:  containerd://df0215f93e3193eaf81281e40cbdaa6ca1136c2ee3268fe3bcb60875f34bdbbf
    Image:         quay.io/cilium/cilium-envoy:v1.34.4-1754895458-68cffdfa568b6b226d70a7ef81fc65dda3b890bf@sha256:247e908700012f7ef56f75908f8c965215c26a27762f296068645eb55450bda2
    Image ID:      quay.io/cilium/cilium-envoy@sha256:247e908700012f7ef56f75908f8c965215c26a27762f296068645eb55450bda2
    Port:          9964/TCP
    Host Port:     9964/TCP
    Command:
      /usr/bin/cilium-envoy-starter
    Args:
      --
      -c /var/run/cilium/envoy/bootstrap-config.json
      --base-id 0
    ...
    Mounts:
      /sys/fs/bpf from bpf-maps (rw)
      /var/run/cilium/envoy/ from envoy-config (ro)
      /var/run/cilium/envoy/artifacts from envoy-artifacts (ro)
      /var/run/cilium/envoy/sockets from envoy-sockets (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-6gsbl (ro)
...
Volumes:
  envoy-sockets:
    Type:          HostPath (bare host directory volume)
    Path:          /var/run/cilium/envoy/sockets
    HostPathType:  DirectoryOrCreate
  envoy-artifacts:
    Type:          HostPath (bare host directory volume)
    Path:          /var/run/cilium/envoy/artifacts
    HostPathType:  DirectoryOrCreate
  envoy-config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      cilium-envoy-config
    Optional:  false
  bpf-maps:
    Type:          HostPath (bare host directory volume)
    Path:          /sys/fs/bpf
    HostPathType:  DirectoryOrCreate

#
ls -al /var/run/cilium/envoy/sockets
total 0
drwxr-xr-x 3 root root 120 Aug 16 17:47 .
drwxr-xr-x 4 root root  80 Aug 16 16:16 ..
srw-rw---- 1 root 1337   0 Aug 16 17:47 access_log.sock
srwxr-xr-x 1 root root   0 Aug 16 16:16 admin.sock
drwxr-xr-x 3 root root  60 Aug 16 16:16 envoy
srw-rw---- 1 root 1337   0 Aug 16 17:47 xds.sock

#
kubectl exec -it -n kube-system ds/cilium-envoy -- ls -al /var/run/cilium/envoy
kubectl exec -it -n kube-system ds/cilium-envoy -- cat /var/run/cilium/envoy/bootstrap-config.json
kubectl exec -it -n kube-system ds/cilium-envoy -- cat /var/run/cilium/envoy/bootstrap-config.json > envoy.json
cat envoy.json | jq

# envoy configmap 설정 내용 확인
kubectl -n kube-system get configmap cilium-envoy-config
kubectl -n kube-system get configmap cilium-envoy-config -o json \
  | jq -r '.data["bootstrap-config.json"]' \
  | jq .
...
{
  "admin": {
    "address": {
      "pipe": {
        "path": "/var/run/cilium/envoy/sockets/admin.sock"
      }
    }
  },
...
    "listeners": [
      {
        "address": {
          "socketAddress": {
            "address": "0.0.0.0",
            "portValue": 9964
...

tree /sys/fs/bpf
/sys/fs/bpf
├── cilium
│   ├── devices
│   │   ├── cilium_host
│   │   │   └── links
│   │   │       ├── cil_from_host
│   │   │       └── cil_to_host
│   │   ├── cilium_net
│   │   │   └── links
│   │   │       └── cil_to_host
│   │   ├── eth0
│   │   │   └── links
│   │   │       ├── cil_from_netdev
│   │   │       └── cil_to_netdev
│   │   └── eth1
│   │       └── links
│   │           ├── cil_from_netdev
│   │           └── cil_to_netdev
│   ├── endpoints
│   │   ├── 1059
│   │   │   └── links
│   │   │       └── cil_from_container
...

#
kubectl get svc,ep -n kube-system cilium-envoy
NAME                   TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
service/cilium-envoy   ClusterIP   None         <none>        9964/TCP   114m

NAME                     ENDPOINTS                                 AGE
endpoints/cilium-envoy   192.168.10.100:9964,192.168.10.101:9964   114m

#
kubectl get svc,ep -n kube-system cilium-ingress
NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
service/cilium-ingress   LoadBalancer   10.96.154.81   <pending>     80:32046/TCP,443:31695/TCP   114m

NAME                       ENDPOINTS              AGE
endpoints/cilium-ingress   192.192.192.192:9999   114m # Cilium → Envoy 간의 제어 채널(Control Plane), 외부 클라이언트가 접근하는 데이터 채널이 아님 by ChatGPT
```

# LB-IPAM 설정 후 확인 : CiliumL2AnnouncementPolicy

```bash
# 현재 L2 Announcement 활성화 상태
cilium config view | grep l2
enable-l2-announcements                           true
enable-l2-neigh-discovery                         false

# 충돌나지 않는지 대역 확인 할 것!
cat << EOF | kubectl apply -f -
apiVersion: "cilium.io/v2" 
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
kubectl get ippool
kubectl get ippools -o jsonpath='{.items[*].status.conditions[?(@.type!="cilium.io/PoolConflict")]}' | jq


# L2 Announcement 정책 설정
cat << EOF | kubectl apply -f -
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: policy1
spec:
  interfaces:
  - eth1
  externalIPs: true
  loadBalancerIPs: true
EOF
```

```bash
# 현재 리더 역할 노드 확인
kubectl -n kube-system get lease | grep "cilium-l2announce"
kubectl -n kube-system get lease/cilium-l2announce-kube-system-cilium-ingress -o yaml | yq

# K8S 클러스터 내부 LB EX-IP로 호출 가능
LBIP=$(kubectl get svc -n kube-system cilium-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $LBIP
arping -i eth1 $LBIP -c 2

# k8s 외부 노드(router)에서 LB EX-IP로 호출 가능 확인
sshpass -p 'vagrant' ssh vagrant@router sudo arping -i eth1 $LBIP -c 2
```

# Ingress HTTP Example : XFF 확인 - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/http/)
The example ingress configuration routes traffic to backend services from the bookinfo demo microservices app from the Istio project.
![[Pasted image 20250817212131.png]]

```bash
# Deploy the Demo App : 공식 문서는 release-1.11 로 ARM CPU 에서 실패한다. 1.26 버전을 높여서 샘플 배포 할 것!
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml

# istio 와 다르게 사이드카 컨테이너가 없다 1/1 , NodePort와 LoadBalancer 서비스 없다.
kubectl get pod,svc,ep

# 
kc describe ingressclasses.networking.k8s.io
kubectl get ingressclasses.networking.k8s.io
NAME     CONTROLLER                     PARAMETERS   AGE
cilium   cilium.io/ingress-controller   <none>       16m


# Basic ingress for istio bookinfo demo application, which can be found in below
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basic-ingress
  namespace: default
spec:
  ingressClassName: cilium
  rules:
  - http:
      paths:
      - backend:
          service:
            name: details
            port:
              number: 9080
        path: /details
        pathType: Prefix
      - backend:
          service:
            name: productpage
            port:
              number: 9080
        path: /
        pathType: Prefix
EOF
```

```bash 
# Adress 는 cilium-ingress LoadBalancer 의 EX-IP
kubectl get svc -n kube-system cilium-ingress
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
cilium-ingress   LoadBalancer   10.96.154.81   192.168.10.211   80:32046/TCP,443:31695/TCP   154m

kubectl get ingress
NAME            CLASS    HOSTS   ADDRESS          PORTS   AGE
basic-ingress   cilium   *       192.168.10.211   80      33s

kc describe ingress
  Host        Path  Backends
  ----        ----  --------
  *           
              /details   details:9080 (172.20.1.59:9080)
              /          productpage:9080 (172.20.1.202:9080)

# 호출 확인
LBIP=$(kubectl get svc -n kube-system cilium-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $LBIP

# 실패하는 호출이 있는가?
curl -so /dev/null -w "%{http_code}\n" http://$LBIP/
curl -so /dev/null -w "%{http_code}\n" http://$LBIP/details/1
curl -so /dev/null -w "%{http_code}\n" http://$LBIP/ratings

# Access the Bookinfo application
curl "http://$LBIP/productpage?u=normal"


# 모니터링
cilium hubble port-forward&
hubble observe -f -t l7
or 
hubble observe -f --identity ingress
...


# router에서 호출
curl -so /dev/null -w "%{http_code}\n" http://$LBIP/
curl -so /dev/null -w "%{http_code}\n" http://$LBIP/details/1

sshpass -p 'vagrant' ssh vagrant@router curl -s http://$LBIP/
sshpass -p 'vagrant' ssh vagrant@router curl -s http://$LBIP/details/1 -v
...
< server: envoy
< date: Sat, 16 Aug 2025 09:57:38 GMT
< content-length: 178
< x-envoy-upstream-service-time: 20
sshpass -p 'vagrant' ssh vagrant@router curl -s http://$LBIP/ratings


# productpage-v1 파드가 배포된 노드 확인 
kubectl get pod -l app=productpage -owide
NAME                              READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
productpage-v1-5b69f9996c-5rrkz   1/1     Running   0          13m   172.20.1.231   k8s-w1   <none>           <none>

# 해당 노드(k8s-w1)에서 veth 인터페이스 정보 확인
PROID=172.20.1.231

ip route |grep $PROID
172.20.1.231 dev lxc80e7403fef4e proto kernel scope link

PROVETH=lxc80e7403fef4e

# ngrep 로 veth 트래픽 캡쳐 : productpage 는 9080 TCP Port 사용
ngrep -tW byline -d $PROVETH '' 'tcp port 9080'


# 외부에서 호출 시도
sshpass -p 'vagrant' ssh vagrant@router curl -s http://$LBIP


# ngrep 로 veth 트래픽 캡쳐 : productpage 는 9080 TCP Port 사용
ngrep -tW byline -d $PROVETH '' 'tcp port 9080'
## igress(envoy) 가 XFF에 client-ip 담고, 목적지 파드로 요청
T 2025/08/16 19:33:06.047280 10.0.2.15:57482 -> 172.20.1.231:9080 [AP] #38
GET / HTTP/1.1.
host: 192.168.10.211.
user-agent: curl/8.5.0.
accept: */*.
x-forwarded-for: 192.168.10.200.
x-forwarded-proto: http.
x-envoy-internal: true.
x-request-id: afa46241-b8ce-4ad4-85de-13da2474b55d.
.

## igress(envoy)로 리턴하는 트래픽
T 2025/08/16 19:33:06.081311 172.20.1.231:9080 -> 10.0.2.15:57482 [AP] #40
HTTP/1.1 200 OK.
Server: gunicorn.
Date: Sat, 16 Aug 2025 10:33:06 GMT.
Connection: keep-alive.
Content-Type: text/html; charset=utf-8.
Content-Length: 2080.
```

![[Pasted image 20250817212218.png]]

# Ingress-Nginx 설치 및 설정 : cilium ingress 와 공존 확인

```bash
# Ingress-Nginx 컨트롤러 설치
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx --create-namespace -n ingress-nginx

# 확인
kubectl get all -n ingress-nginx
kc describe svc -n ingress-nginx ingress-nginx-controller
kubectl get svc -n ingress-nginx
kubectl get ingressclasses.networking.k8s.io
NAME     CONTROLLER                     PARAMETERS   AGE
cilium   cilium.io/ingress-controller   <none>       18m
nginx    k8s.io/ingress-nginx           <none>       22s


# ingress 설정
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webpod-ingress-nginx
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.webpod.local
    http:
      paths:
      - backend:
          service:
            name: webpod
            port:
              number: 80
        path: /
        pathType: Prefix
EOF
```

```bash
# ingress LB EX-IP 할당 까지 다소 시간 소요..
kubectl get ingress -w
NAME                   CLASS    HOSTS                ADDRESS          PORTS   AGE
basic-ingress          cilium   *                    192.168.10.211   80      3m19s
webpod-ingress-nginx   nginx    nginx.webpod.local   192.168.10.212   80      11m

#
LB2IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

curl $LB2IP
curl -H "Host: nginx.webpod.local" $LB2IP
curl -H "Host: nginx.webpod.local" $LB2IP

sshpass -p 'vagrant' ssh vagrant@router "curl -s -H 'Host: nginx.webpod.local' $LB2IP"
sshpass -p 'vagrant' ssh vagrant@router "curl -s -H 'Host: nginx.webpod.local' $LB2IP"
```

# dedicated mode
```bash
# Basic ingress for istio bookinfo demo application, which can be found in below
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webpod-ingress
  namespace: default
  annotations:
    ingress.cilium.io/loadbalancer-mode: dedicated
spec:
  ingressClassName: cilium
  rules:
  - http:
      paths:
      - backend:
          service:
            name: webpod
            port:
              number: 80
        path: /
        pathType: Prefix
EOF
```

```bash
#
kc describe ingress webpod-ingress
kubectl get ingress
NAME                   CLASS    HOSTS                ADDRESS          PORTS   AGE
basic-ingress          cilium   *                    192.168.10.211   80      8m26s
webpod-ingress         cilium   *                    192.168.10.213   80      8s
webpod-ingress-nginx   nginx    nginx.webpod.local   192.168.10.212   80      16m

kubectl get svc,ep cilium-ingress-webpod-ingress
NAME                                    TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
service/cilium-ingress-webpod-ingress   LoadBalancer   10.96.175.13   192.168.10.213   80:31067/TCP,443:30783/TCP   27s

NAME                                      ENDPOINTS              AGE
endpoints/cilium-ingress-webpod-ingress   192.192.192.192:9999   27s

# LB EX-IP에 대한 L2 Announcement 의 Leader 노드 확인
kubectl get lease -n kube-system | grep ingress

# webpod 파드 IP 확인
kubectl get pod -l app=webpod -owide
NAME                      READY   STATUS    RESTARTS   AGE   IP             NODE      NOMINATED NODE   READINESS GATES
webpod-7d9dcd8859-7v76g   1/1     Running   0          16s   172.20.0.112   k8s-ctr   <none>           <none>
webpod-7d9dcd8859-kpjrj   1/1     Running   0          20s   172.20.1.38    k8s-w1    <none>           <none>

# k8c-ctr, k8s-w1 노드에서 파드 IP에 veth 찾기(ip -c route) 이후 ngrep 로 각각 트래픽 캡쳐
WPODVETH=lxce95c39b2b4c4
ngrep -tW byline -d $WPODVETH '' 'tcp port 80'

# router 에서 호출 확인
LB2IP=$(kubectl get svc cilium-ingress-webpod-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
sshpass -p 'vagrant' ssh vagrant@router curl -s http://$LB2IP
Hostname: webpod-7d9dcd8859-kpjrj
IP: 127.0.0.1
IP: ::1
IP: 172.20.1.38  # webpod 파드 IP
IP: fe80::bce5:35ff:fee8:619e
RemoteAddr: 10.0.2.15:51984  # webpod 인입 시 S.IP : 마치 L2 Leader 노드에 webpod로 전달되어, 소스IP가 해당 노드의 첫 번째 NIC IP, 
GET / HTTP/1.1
Host: 192.168.10.212
User-Agent: curl/8.5.0
Accept: */*
X-Envoy-Internal: true
X-Forwarded-For: 192.168.10.200
X-Forwarded-Proto: http
X-Request-Id: 292040af-f112-4d04-9281-730da5a513d1

sshpass -p 'vagrant' ssh vagrant@router curl -s http://$LB2IP
Hostname: webpod-7d9dcd8859-7v76g
IP: 127.0.0.1
IP: ::1
IP: 172.20.0.112  # webpod 파드 IP
IP: fe80::607a:a1ff:fe34:205b
RemoteAddr: 172.20.1.35:46067  # webpod 인입 시 S.IP : L2 Leader 노드(k8s-w1)에서 다른 노드에 파드로 전달되어, ingress 예약IP로 SNAT
GET / HTTP/1.1
Host: 192.168.10.212
User-Agent: curl/8.5.0
Accept: */*
X-Envoy-Internal: true
X-Forwarded-For: 192.168.10.200
X-Forwarded-Proto: http
X-Request-Id: dd65f484-8d20-4a4c-8eac-ed8666f64bdf
```

# 도전과제1-1 Ingress Host network mode - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/ingress/#host-network-mode)

^d88995


# Ingress and Network Policy Example - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/ingress-and-network-policy/)
```bash
# 클러스터 전체(모든 네임스페이스)에 적용되는 정책 : 참고로 아래 정책 적용 후 Hubble-ui 로 접속 불가!
cat << EOF | kubectl apply -f -
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "external-lockdown"
spec:
  description: "Block all the traffic originating from outside of the cluster"
  endpointSelector: {}
  ingress:
  - fromEntities:
    - cluster
EOF
```
```bash
kubectl get ciliumclusterwidenetworkpolicy


#
curl --fail -v http://"$LBIP"/details/1
< HTTP/1.1 403 Forbidden

# 
hubble observe -f --identity ingress

## k8s-ctr 에서 curl 실행 시
Aug 16 14:02:01.011: 127.0.0.1:58328 (ingress) -> 127.0.0.1:14485 (world) http-request DROPPED (HTTP/1.1 GET http://192.168.10.211/details/1)
Aug 16 14:02:01.011: 127.0.0.1:58328 (ingress) <- 127.0.0.1:14485 (world) http-response FORWARDED (HTTP/1.1 403 0ms (GET http://192.168.10.211/details/1))

## router 에서 curl 실행 시
Aug 16 14:03:35.580: 192.168.10.200:56904 (ingress) -> kube-system/cilium-ingress:80 (world) http-request DROPPED (HTTP/1.1 GET http://192.168.10.211/)
Aug 16 14:03:35.580: 192.168.10.200:56904 (ingress) <- kube-system/cilium-ingress:80 (world) http-response FORWARDED (HTTP/1.1 403 0ms (GET http://192.168.10.211/))

# 스터디 시간(영상 녹화)에서 누락된 내용으로 아래 리소스 생성 후 진행하시면 됩니다!
cat << EOF | kubectl apply -f -
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "allow-cidr"
spec:
  description: "Allow all the traffic originating from a specific CIDR"
  endpointSelector:
    matchExpressions:
    - key: reserved:ingress
      operator: Exists
  ingress:
  - fromCIDRSet:
    # Please update the CIDR to match your environment
    - cidr: 192.168.10.200/32
    - cidr: 127.0.0.1/32
EOF
```

```bash

# 요청 성공! : k8s-ctr , router 모두 가능
curl --fail -v http://"$LBIP"/details/1
sshpass -p 'vagrant' ssh vagrant@router "curl -s --fail -v http://"$LBIP"/details/1"


# Default Deny Ingress Policy : DNS쿼리와 kube-system내의 파드 제외 to deny all traffic by default
cat << EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "default-deny"
spec:
  description: "Block all the traffic (except DNS) by default"
  egress:
  - toEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: kube-system
        k8s-app: kube-dns
    toPorts:
    - ports:
      - port: '53'
        protocol: UDP
      rules:
        dns:
        - matchPattern: '*'
  endpointSelector:
    matchExpressions:
    - key: io.kubernetes.pod.namespace
      operator: NotIn
      values:
      - kube-system
EOF
```

```bash
kubectl get ciliumclusterwidenetworkpolicy


# 요청 
curl --fail -v http://"$LBIP"/details/1
sshpass -p 'vagrant' ssh vagrant@router "curl -s --fail -v http://"$LBIP"/details/1"


# ingress 를 통해서 인입 시 허용
cat << EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-ingress-egress
spec:
  description: "Allow all the egress traffic from reserved ingress identity to any endpoints in the cluster"
  endpointSelector:
    matchExpressions:
    - key: reserved:ingress
      operator: Exists
  egress:
  - toEntities:
    - cluster
EOF
```

```bash
kubectl get ciliumclusterwidenetworkpolicy

#
curl --fail -v http://"$LBIP"/details/1
sshpass -p 'vagrant' ssh vagrant@router "curl -s --fail -v http://"$LBIP"/details/1"


# 정책 삭제
kubectl delete CiliumClusterwideNetworkPolicy --all
```

# Ingress Path Types Example - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/path-types/)

```bash
# Apply the base definitions
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/main/examples/kubernetes/servicemesh/ingress-path-types.yaml

# 확인
kubectl get -f https://raw.githubusercontent.com/cilium/cilium/main/examples/kubernetes/servicemesh/ingress-path-types.yaml

# Apply the Ingress
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/main/examples/kubernetes/servicemesh/ingress-path-types-ingress.yaml

# 확인
kc describe ingress multiple-path-types
kc get ingress multiple-path-types -o yaml
spec:
  ingressClassName: cilium
  rules:
  - host: pathtypes.example.com
    http:
      paths:
      - backend:
          service:
            name: exactpath
            port:
              number: 80
        path: /exact
        pathType: Exact
      - backend:
          service:
            name: prefixpath
            port:
              number: 80
        path: /
        pathType: Prefix
      - backend:
          service:
            name: prefixpath2
            port:
              number: 80
        path: /prefix
        pathType: Prefix
      - backend:
          service:
            name: implpath
            port:
              number: 80
        path: /impl
        pathType: ImplementationSpecific
      - backend:
          service:
            name: implpath2
            port:
              number: 80
        path: /impl.+
        pathType: ImplementationSpecific

# 호출 확인
export PATHTYPE_IP=`k get ing multiple-path-types -o json | jq -r '.status.loadBalancer.ingress[0].ip'`
curl -s -H "Host: pathtypes.example.com" http://$PATHTYPE_IP/ | jq


# 파드명 이름 확인
kubectl get pod | grep path
exactpath-7488f8c6c6-pd4pc        1/1     Running   0          7m25s
implpath-7d8bf85676-f7k5s         1/1     Running   0          7m25s
implpath2-56c97c8556-mtnnm        1/1     Running   0          7m25s
prefixpath-5d6b989d4-w4qv2        1/1     Running   0          7m25s
prefixpath2-b7c7c9568-mkv8d       1/1     Running   0          7m25s


# Should show prefixpath
curl -s -H "Host: pathtypes.example.com" http://$PATHTYPE_IP/ | grep -E 'path|pod'
 "path": "/",
 "host": "pathtypes.example.com",
 "pod": "prefixpath-5d6b989d4-w4qv2"


# Should show exactpath
curl -s -H "Host: pathtypes.example.com" http://$PATHTYPE_IP/exact | grep -E 'path|pod'
 "path": "/exact",
 "host": "pathtypes.example.com",
 "pod": "exactpath-7488f8c6c6-pd4pc"


# Should show prefixpath2
curl -s -H "Host: pathtypes.example.com" http://$PATHTYPE_IP/prefix | grep -E 'path|pod'
 "path": "/prefix",
 "host": "pathtypes.example.com",
 "pod": "prefixpath2-b7c7c9568-mkv8d"


# Should show implpath
curl -s -H "Host: pathtypes.example.com" http://$PATHTYPE_IP/impl | grep -E 'path|pod'
 "path": "/impl",
 "host": "pathtypes.example.com",
 "pod": "implpath-7d8bf85676-f7k5s"


# Should show implpath2
curl -s -H "Host: pathtypes.example.com" http://$PATHTYPE_IP/implementation | grep -E 'path|pod'
 "path": "/implementation",
 "host": "pathtypes.example.com",
 "pod": "implpath2-56c97c8556-mtnnm"


# 삭제
kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/main/examples/kubernetes/servicemesh/ingress-path-types.yaml
kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/main/examples/kubernetes/servicemesh/ingress-path-types-ingress.yaml
```

# 도전과제1-2 ~~Ingress gRPC Example~~ : arm64 CPU 미지원 - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/grpc/)

^5fd0f5

- Sample demo app - [Github](https://github.com/GoogleCloudPlatform/microservices-demo)
- grpCurl - [Github](https://github.com/fullstorydev/grpcurl)


# Ingress Example with TLS Termination - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/tls-termination/)
- Create TLS Certificate and Private Key : mkcert - [Github](https://github.com/FiloSottile/mkcert)
    - mkcert is a simple tool for making locally-trusted development certificates. It requires no configuration.
```bash
# For demonstration purposes we will use a TLS certificate signed by a made-up, self-signed certificate authority (CA). 
# One easy way to do this is with mkcert. We want a certificate that will validate bookinfo.cilium.rocks and hipstershop.cilium.rocks, as these are the host names used in this example.
apt install mkcert -y
mkcert -h

#
mkcert '*.cilium.rocks'

#
ls -l *.pem
-rw------- 1 root root 1708 Aug 17 12:11 _wildcard.cilium.rocks-key.pem
-rw-r--r-- 1 root root 1452 Aug 17 12:11 _wildcard.cilium.rocks.pem

#
openssl x509 -in _wildcard.cilium.rocks.pem -text -noout
        Issuer: O = mkcert development CA, OU = root@k8s-ctr, CN = mkcert root@k8s-ctr
        Validity
            Not Before: Aug 17 03:11:30 2025 GMT
            Not After : Nov 17 03:11:30 2027 GMT
        Subject: O = mkcert development certificate, OU = root@k8s-ctr
        ...
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication
            X509v3 Authority Key Identifier: 
                7E:20:AF:5F:00:4A:C8:6B:D6:44:C5:4F:32:C0:ED:FB:3C:0D:30:00
            X509v3 Subject Alternative Name: 
                DNS:*.cilium.rocks

openssl rsa -in _wildcard.cilium.rocks-key.pem -text -noout
Private-Key: (2048 bit, 2 primes)
modulus:
    00:cb:35:d6:f0:e2:77:41:4b:ea:39:ea:06:bc:5d:
...


# Mkcert created a key (_wildcard.cilium.rocks-key.pem) and a certificate (_wildcard.cilium.rocks.pem) that we will use for the Gateway service.
# Create a Kubernetes TLS secret with this key and certificate:
kubectl create secret tls demo-cert --key=_wildcard.cilium.rocks-key.pem --cert=_wildcard.cilium.rocks.pem
kubectl get secret demo-cert -o json | jq

```

## Deploy the Ingress
```bash
#
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: default
spec:
  ingressClassName: cilium
  rules:
  - host: webpod.cilium.rocks
    http:
      paths:
      - backend:
          service:
            name: webpod
            port:
              number: 80
        path: /
        pathType: Prefix
  - host: bookinfo.cilium.rocks
    http:
      paths:
      - backend:
          service:
            name: details
            port:
              number: 9080
        path: /details
        pathType: Prefix
      - backend:
          service:
            name: productpage
            port:
              number: 9080
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - webpod.cilium.rocks
    - bookinfo.cilium.rocks
    secretName: demo-cert
EOF
```

```bash
#    
kubectl get ingress tls-ingress
NAME          CLASS    HOSTS                                       ADDRESS          PORTS     AGE
tls-ingress   cilium   webpod.cilium.rocks,bookinfo.cilium.rocks   192.168.10.211   80, 443   8s
```

## Make requests
```bash 
# 시스템(OS) 신뢰 저장소에 CA 정보 확인
cat /etc/ssl/certs/ca-certificates.crt
ls -al /etc/ssl/certs/ca-certificates.crt
-rw-r--r-- 1 root root 220952 Aug 17 12:20 /etc/ssl/certs/ca-certificates.crt

# Install the Mkcert CA into your system so cURL can trust it:
# mkcert -install은 “내 로컬에서 만든 인증서를 시스템이 믿도록” 환경을 꾸며주는 명령.
mkcert -install

1. 로컬 CA(자체 루트 인증기관) 생성
# 아직 없으면 로컬 CA 인증서와 개인키를 만듭니다.
# 파일은 $(mkcert -CAROOT)가 가리키는 사용자 데이터 디렉터리에 저장돼요(예: rootCA.pem, rootCA-key.pem). 이 위치는 mkcert -CAROOT로 확인합니다. 
## CA 저장 위치 확인
mkcert -CAROOT
/root/.local/share/mkcert

ls "$(mkcert -CAROOT)"
rootCA-key.pem  rootCA.pem


2. 시스템(OS) 신뢰 저장소에 CA를 등록
# 배포판에 맞는 도구(예: Debian/Ubuntu의 update-ca-certificates, RHEL/Fedora의 update-ca-trust, Arch의 trust)를 사용해 루트 CA 인증서를 시스템 신뢰 저장소에 넣습니다.
# 이렇게 하면 OpenSSL/GnuTLS를 쓰는 대부분의 CLI가 이 CA로 서명된 서버 인증서를 신뢰합니다. (curl도 포함)
ls -al /etc/ssl/certs/ca-certificates.crt
-rw-r--r-- 1 root root 220948 Aug 16 01:04 /etc/ssl/certs/ca-certificates.crt

## 맨 하단에 인증서만 파일로 만들어서 디코딩!
tail -n 50 /etc/ssl/certs/ca-certificates.crt
vi 1.pem
openssl x509 -in 1.pem -text -noout
        Issuer: O = mkcert development CA, OU = root@k8s-ctr, CN = mkcert root@k8s-ctr
        Validity
            Not Before: Aug 17 03:09:12 2025 GMT
            Not After : Aug 17 03:09:12 2035 GMT
        Subject: O = mkcert development CA, OU = root@k8s-ctr, CN = mkcert root@k8s-ctr
        ...
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:0



3. NSS(브라우저) 신뢰 저장소에 등록
# Linux에서는 Firefox/Chromium이 쓰는 NSS 데이터베이스에도 CA를 넣습니다(사전에 certutil 설치 필요). Firefox는 브라우저 재시작이 필요합니다


# Now let's make a request to the Gateway:
# The data should be properly retrieved, using HTTPS (and thus, the TLS handshake was properly achieved).
# In the next challenge, we will see how to use Gateway API for general TLS traffic.
kubectl get ingress tls-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
LBIP=$(kubectl get ingress tls-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

curl -s --resolve bookinfo.cilium.rocks:443:${LBIP} https://bookinfo.cilium.rocks/details/1 | jq
  
curl -s --resolve webpod.cilium.rocks:443:${LBIP}   https://webpod.cilium.rocks/ -v
...
*  CAfile: /etc/ssl/certs/ca-certificates.crt
*  CApath: /etc/ssl/certs
...
* Server certificate:
*  subject: O=mkcert development certificate; OU=root@k8s-ctr
*  start date: Aug 17 03:11:30 2025 GMT
*  expire date: Nov 17 03:11:30 2027 GMT
*  subjectAltName: host "webpod.cilium.rocks" matched cert's "*.cilium.rocks"
*  issuer: O=mkcert development CA; OU=root@k8s-ctr; CN=mkcert root@k8s-ctr
...
Hostname: webpod-697b545f57-nvf6n
IP: 127.0.0.1
IP: ::1
IP: 172.20.0.208
IP: fe80::58b2:8ff:fe48:6959
RemoteAddr: 10.0.2.15:34726
GET / HTTP/1.1
Host: webpod.cilium.rocks
User-Agent: curl/8.5.0
Accept: */*
X-Envoy-Internal: true
X-Forwarded-For: 10.0.2.15
X-Forwarded-Proto: https
X-Request-Id: 11e3e44c-f1aa-4939-bec9-06cabb7c76b3
```

# 도전과제1-3 Defaults certificate for Ingresses - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/tls-default-certificate/)

^cb008f


```bash
- Ingress 삭제 `kubectl delete ingress basic-ingress tls-ingress webpod-ingress`
```