![[Pasted image 20250726005523.png]]
- 스타워즈에서 영감을 받은 예제에서는 데스스타, 타이파이터, 엑스윙의 세 가지 마이크로서비스 애플리케이션이 있습니다.
- 데스스타는 포트 80에서 HTTP 웹서비스를 실행하며, 이 서비스는 두 개의 포드 복제본에 걸쳐 데스스타에 대한 요청을 로드 밸런싱하는 Kubernetes 서비스로 노출됩니다.
- 데스스타 서비스는 제국의 우주선에 착륙 서비스를 제공하여 착륙 포트를 요청할 수 있도록 합니다.
- 타이파이터 포드는 일반적인 제국 선박의 착륙 요청 클라이언트 서비스를 나타내며, 엑스윙은 동맹 선박의 유사한 서비스를 나타냅니다.
- 데스스타 착륙 서비스에 대한 접근 제어를 위한 다양한 보안 정책을 테스트할 수 있도록 존재합니다.

# 배포 (Restart)
```bash
#
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/http-sw-app.yaml
service/deathstar created
deployment.apps/deathstar created
pod/tiefighter created
pod/xwing created

# 파드 라벨 labels 확인
kubectl get pod --show-labels
NAME                        READY   STATUS    RESTARTS   AGE   LABELS
deathstar-8c4c77fb7-9klws   1/1     Running   0          29s   app.kubernetes.io/name=deathstar,class=deathstar,org=empire,pod-template-hash=8c4c77fb7
deathstar-8c4c77fb7-kkwds   1/1     Running   0          29s   app.kubernetes.io/name=deathstar,class=deathstar,org=empire,pod-template-hash=8c4c77fb7
tiefighter                  1/1     Running   0          29s   app.kubernetes.io/name=tiefighter,class=tiefighter,org=empire
xwing                       1/1     Running   0          29s   app.kubernetes.io/name=xwing,class=xwing,org=alliance

kubectl get deploy,svc,ep deathstar
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deathstar   2/2     2            2           114s

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/deathstar   ClusterIP   10.96.154.223   <none>        80/TCP    114s

NAME                  ENDPOINTS                      AGE
endpoints/deathstar   172.20.1.34:80,172.20.2.1:80   114s

#
kubectl get ciliumendpoints.cilium.io -A
kubectl get ciliumidentities.cilium.io

# in a multi-node installation, only the ones running on the same node will be listed
kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium endpoint list
c0 endpoint list
c1 endpoint list
c2 endpoint list # 현재 ingress/egress 에 정책(Policy) 없음! , Labels 정보 확인
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS (source:key[=value])
...
1579       Disabled           Disabled          318        k8s:app.kubernetes.io/name=deathstar                                                172.20.2.1    ready   
                                                           k8s:class=deathstar                                                                                       
                                                           k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default                                    
                                                           k8s:io.cilium.k8s.policy.cluster=default                                                                  
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                           
                                                           k8s:io.kubernetes.pod.namespace=default                                                                   
                                                           k8s:org=empire   

```

# 현재 상태 Check
[[Cilium Agent 단축키 지정]] 선행 
- 데스스타 서비스의 관점에서 보면, org= empire 라벨이 부착된 선박만 연결하여 착륙을 요청할 수 있습니다.
- 우리는 규칙을 시행하지 않기 때문에 Xwing과 타이파이터 모두 착륙을 요청할 수 있습니다. 이를 테스트하려면 아래 명령을 사용하세요.
- 
```bash
# 아래 출력에서 xwing 와 tiefighter 의 IDENTITY 메모
c1 endpoint list | grep -iE 'xwing|tiefighter|deathstar'
c2 endpoint list | grep -iE 'xwing|tiefighter|deathstar'
XWINGID=1723
TIEFIGHTERID=39615
DEATHSTARID=10151

# 모니터링 준비 : 터미널 3개, 단축키 설정
## 각각 monitor 확인
c0 monitor -v -v
c1 monitor -v -v
c2 monitor -v -v

# 모니터링 준비 : 터미널 1개
hubble observe -f

XWINGID=1723
TIEFIGHTERID=39615
DEATHSTARID=10151

hubble observe -f --from-identity $XWINGID
hubble observe -f --protocol udp --from-identity $XWINGID
hubble observe -f --protocol tcp --from-identity $XWINGID

hubble observe -f --protocol tcp --from-identity $DEATHSTARID


# 호출 시도 1
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
while true; do kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing ; sleep 5 ; done

# 호출 시도 2
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
while true; do kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing ; sleep 5 ; done

## 모니터링
hubble observe -f --protocol tcp --from-identity $TIEFIGHTERID
hubble observe -f --protocol tcp --from-identity $DEATHSTARID
```

## XWING -> DEATHSTAR
hubble observe -f --from-identity $XWINGID
hubble observe -f --protocol udp --from-identity $XWINGID
hubble observe -f --protocol tcp --from-identity $XWINGID
hubble observe -f --protocol tcp --from-identity $DEATHSTARID

![[Pasted image 20250726201311.png]]

![[Pasted image 20250726011819.png]]
![[Pasted image 20250726011832.png]]
- Cilium을 사용할 때 보안 정책을 정의할 때 엔드포인트 IP 주소는 중요하지 않습니다. 대신 **포드에 할당된 레이블**을 사용하여 **보안 정책을 정의**할 수 있습니다. 정책은 클러스터 내에서 실행 중이거나 실행 중인 위치에 관계없이 레이블을 기반으로 올바른 포드에 적용됩니다.
- **데스스타 착륙 요청**을 라벨이 있는 선박(org=empire)으로만 **제한**하는 **기본 정책**부터 시작하겠습니다. 이렇게 하면 org=empire 라벨이 없는 선박은 데스스타 서비스와 연결조차 할 수 없습니다. 이 정책은 IP 프로토콜(네트워크 계층 3)과 TCP 프로토콜(네트워크 계층 4)에만 적용되는 간단한 정책이므로 흔히 **L3/L4 네트워크 보안 정책**이라고 합니다.
- 참고: 실리움은 **상태별 연결 추적**을 수행합니다. 이는 정책이 프론트엔드가 백엔드에 도달할 수 있도록 허용하면, 동일한 TCP/UDP 연결 내에서 백엔드 응답의 일부인 모든 **필수 응답 패킷이 자동**으로 프론트엔드에 도달하도록 **허용**한다는 것을 의미합니다. → 리턴 패킷 자동 허용!

```bash
# CiliumNetworkPolicy
## CiliumNetworkPolicys는 "endpointSelector"를 사용하여 팟 레이블에서 정책이 적용되는 소스와 목적지를 식별합니다. 
## 아래 정책은 TCP 포트 80에서 레이블(org=empire)이 있는 모든 팟에서 레이블(org=empire, class=deathstar)이 있는 데스스타 팟으로 전송되는 트래픽을 화이트리스트로 작성합니다.
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L3-L4 policy to restrict deathstar access to empire ships only"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/sw_l3_l4_policy.yaml
kubectl get cnp
kubectl get cnp -o json | jq

# 모니터링
hubble observe -f --type drop

# 호출 시도 1 
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing --connect-timeout 2


# 모니터링 
hubble observe -f --protocol tcp --from-identity $DEATHSTARID

# 호출 시도 2
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

```

## Xwing -> Deathstar
![[스크린샷 2025-07-26 오후 8.18.43.png]]

![[Pasted image 20250726012037.png]]

![[스크린샷 2025-07-26 오후 8.22.03.png]]
![[스크린샷 2025-07-26 오후 8.21.12.png]]
```bash
# deathstar 에 ingress 에 policy 활성화 확인
c0 endpoint list
c1 endpoint list
c2 endpoint list
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS (source:key[=value]) 
...
1579       Enabled            Disabled          318        k8s:app.kubernetes.io/name=deathstar                                                172.20.2.1    ready   
                                                           k8s:class=deathstar                                                                                       
                                                           k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default                                    
                                                           k8s:io.cilium.k8s.policy.cluster=default                                                                  
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                           
                                                           k8s:io.kubernetes.pod.namespace=default                                                                   
                                                           k8s:org=empire  
                                                           
#
kc describe cnp rule1

```
![[스크린샷 2025-07-26 오후 8.23.26.png]]

[[Cilium Packet Flow]] 참고 

![[Pasted image 20250726012307.png]]
Life of a Packet : L7 동작 처리는 cilium-envoy 데몬셋이 담당

```bash
#
kubectl get ds -n kube-system
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
cilium         3         3         3       3            3           kubernetes.io/os=linux   3h38m
cilium-envoy   3         3         3       3            3           kubernetes.io/os=linux   3h38m

kubectl get pod -n kube-system -l k8s-app=cilium-envoy -owide
NAME                 READY   STATUS    RESTARTS   AGE     IP               NODE      NOMINATED NODE   READINESS GATES
cilium-envoy-8427n   1/1     Running   0          3h38m   192.168.10.100   k8s-ctr   <none>           <none>
cilium-envoy-8rzsb   1/1     Running   0          3h36m   192.168.10.101   k8s-w1    <none>           <none>
cilium-envoy-vkqw6   1/1     Running   0          3h35m   192.168.10.102   k8s-w2    <none>           <none>

#
kc describe ds -n kube-system cilium-envoy
...
    Mounts:
      /sys/fs/bpf from bpf-maps (rw)
      /var/run/cilium/envoy/ from envoy-config (ro)
      /var/run/cilium/envoy/artifacts from envoy-artifacts (ro)
      /var/run/cilium/envoy/sockets from envoy-sockets (rw
   ...
   envoy-config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      cilium-envoy-config
    Optional:  false
...


kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- ss -xnp | grep -i -envoy
u_str ESTAB 0      0                                                  /var/run/cilium/envoy/sockets/xds.sock 32644            * 32610 users:(("cilium-agent",pid=1,fd=64))
u_str ESTAB 0      0                                                /var/run/cilium/envoy/sockets/admin.sock 29697            * 29345
u_str ESTAB 0      0                                                /var/run/cilium/envoy/sockets/admin.sock 29340            * 28672

kc describe cm -n kube-system cilium-envoy-config
...
Data
====
bootstrap-config.json:
----
{"admin":{"address":{"pipe":{"path":"/var/run/cilium/envoy/sockets/admin.sock"}}}...
...

```

![[Pasted image 20250726013549.png]]

- 위의 간단한 시나리오에서는 tiefighter / xwing에게 데스스타 API에 대한 전체 액세스 권한을 부여하거나 아예 액세스 권한을 부여하지 않는 것으로 충분했습니다. 그러나 마이크로서비스 간에 가장 강력한 보안(즉, 최소 권한 격리를 강제하는 것)을 제공하기 위해서는 데스스타 API를 호출하는 각 서비스가 **합법적인 운영에 필요한 HTTP 요청 세트만 수행**하도록 제한해야 합니다.
- 예를 들어, 데스스타 서비스가 임의의 제국 선박이 **호출해서는 안 되는 일부 유지보수 API를 노출**한다고 가정해 보겠습니다.

```bash
# 모니터링 >> Layer3/4 에서는 애플리케이션 상태를 확인 할 수 없음!
hubble observe -f --protocol tcp --from-identity $DEATHSTARID

# 호출해서는 안 되는 일부 유지보수 API를 노출
kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
```
## tiefighter -> deathstar
* Layer L3/L4 단에서 애플리케이션에 이슈가 생겼으나, 트래픽 자체는 넘어감 
![[Pasted image 20250726204545.png]]
![[Pasted image 20250726204503.png]]


Cilium은 HTTP 계층(즉, L7) 정책을 적용하여 타이파이터가 도달할 수 있는 URL을 제한할 수 있습니다. 다음은 타이파이터를 POST /v1/요청-랜딩 API 호출만 수행하도록 제한하고 다른 모든 호출(PUT /v1/배기포트 포함)은 허용하지 않는 정책 파일의 예입니다.

```bash
# 기존 rule1 정책을 업데이트 해서 사용
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L7 policy to restrict access to specific HTTP call"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/v1/request-landing"

# Update the existing rule to apply L7-aware policy to protect deathstar using:
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/sw_l3_l4_l7_policy.yaml
kubectl get cnp
kc describe cnp
c0 policy get


# 모니터링 : 파드 이름 지정
hubble observe -f --pod deathstar --protocol http
Jul 20 01:28:02.184: default/tiefighter:59020 (ID:19274) -> default/deathstar-8c4c77fb7-9klws:80 (ID:318) http-request FORWARDED (HTTP/1.1 POST http://deathstar.default.svc.cluster.local/v1/request-landing)
Jul 20 01:28:02.190: default/tiefighter:59020 (ID:19274) <- default/deathstar-8c4c77fb7-9klws:80 (ID:318) http-response FORWARDED (HTTP/1.1 200 6ms (POST http://deathstar.default.svc.cluster.local/v1/request-landing))

# We can now re-run the same test as above, but we will see a different outcome
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

```
![[Pasted image 20250726013725.png]]

```bash
# 모니터링 : 파드 이름 지정
hubble observe -f --pod deathstar --verdict DROPPED
Jul 20 01:31:21.248: default/tiefighter:44734 (ID:19274) -> default/deathstar-8c4c77fb7-9klws:80 (ID:318) http-request DROPPED (HTTP/1.1 PUT http://deathstar.default.svc.cluster.local/v1/exhaust-port)

혹은
c1 monitor -v --type l7
<- Request http from 3845 ([k8s:app.kubernetes.io/name=tiefighter k8s:class=tiefighter k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=default k8s:io.kubernetes.pod.namespace=default k8s:org=empire]) to 871 ([k8s:app.kubernetes.io/name=deathstar k8s:class=deathstar k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=default k8s:io.kubernetes.pod.namespace=default k8s:org=empire]), identity 19274->318, verdict Denied PUT http://deathstar.default.svc.cluster.local/v1/exhaust-port => 0
<- Response http to 3845 ([k8s:app.kubernetes.io/name=tiefighter k8s:class=tiefighter k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=default k8s:io.kubernetes.pod.namespace=default k8s:org=empire]) from 871 ([k8s:app.kubernetes.io/name=deathstar k8s:class=deathstar k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=default k8s:io.kubernetes.pod.namespace=default k8s:org=empire]), identity 318->19274, verdict Forwarded PUT http://deathstar.default.svc.cluster.local/v1/exhaust-port => 403
c2 monitor -v --type l7


# We can now re-run the same test as above, but we will see a different outcome
kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Access denied



# 모니터링 : 파드 이름 지정
hubble observe -f --pod xwing

# 호출 시도 : 위와 아래 실행 종료의 차이점을 이해해보자!
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing --connect-timeout 2
```

![[Pasted image 20250726013756.png]]

# 왜 차이점이 발생하는지에 대해 
* Cilium의 정책 (CNP) 를 확인해보면 
```bash 
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
meta
  name: "rule1"
spec:
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/v1/request-landing"
```
* tiefighter -> deathstar : PUT /v1/exhaust-port 요청 -> 403 Access Denied (http-request Dropped, 정책에 의한 거절)
* xwing -> deathstar : POST /v1/request-landing 요청 -> SYN 패킷부터 정책 거부, 실제 연결 실패 (policy-verdict:none INGRESS DENIED, Policy denied DROPPED)

## 결론 
* deathstar pod에 붙은 policy는, org: empire로 라벨링된 source만, 80포트로 HTTP POST /v1/request-landing 요청만 허용한다는 의미
* 이외의 모든 트래픽(다른 HTTP 메소드, 다른 path, port, or 라벨 미일치)은 거부

* tiefighter 요청 (`PUT /v1/exhaust-port`)
	* tiefighter가 deathstar로 PUT /v1/exhaust-port를 보냄
	* 정책에 “POST /v1/request-landing”만 허용되어 있으므로 이 요청은 L7(HTTP) 레벨에서 프록시(Envoy 등)에서 탐지 후 http-request DROP → HTTP/1.1 403 (Access denied) 반환
	* 이때 TCP 레벨 connection 자체는 허용 (policy-verdict:L3-L4 INGRESS ALLOWED), 단, HTTP 프록시에서 L7 기준으로 DROP
* xwing 요청 (`POST /v1/request-landing`)
	* xwing이 deathstar로 POST /v1/request-landing 시도
	* 하지만 fromEndpoints:가 org: empire 라벨만 허용하는데, xwing pod가 이 라벨이 없다면 모든 트래픽이 L3/L4 레벨에서 REJECT
	* policy-verdict:none INGRESS DENIED (TCP Flags: SYN)
	* Policy denied DROPPED (TCP Flags: SYN)
	* 즉, TCP 세션 수립 자체가 안 됨 (SYN부터 차단)
	* 연결시도하다가 2초(커넥트 타임아웃)만에 실패(exit code 28) → 실질적으로 deathstar HTTP 서버에 접근 불가
![[스크린샷 2025-07-26 오후 9.22.09.png]]
# 리소스 삭제
```bash
# 다음 실습을 위해 리소스 삭제
kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/http-sw-app.yaml
kubectl delete cnp rule1

# 삭제 확인
kubectl get cnp

```