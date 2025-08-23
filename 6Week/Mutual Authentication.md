# 9.1.4 모놀리스와 마이크로서비스의 보안 비교 Comparison of security in monoliths and microservices

- 마이크로서비스와 모놀리스 모두 최종 사용자 및 **서비스** 간 **인증**과 **인가**를 구현해야 한다.
- 그러나 마이크로서비스에는 보호해야 하는, 네트워크를 오가는 커넥션과 요청이 훨씬 더 많다.
- 반면 모놀리스는 커넥션이 더 적고, 보통은 가상머신 혹은 물리 머신 같은 더 정적인 인프라에서 실행된다.
- **정적인 인프라**에서 실행하면 **(고정) IP 주소**를 **ID 확인 근거**로 심기 좋으며, 덕분에 인증용 인증서에서 흔하게 사용한다. (네트워크 방화벽 규칙에도 사용한다)
- 그림 9.1은 IP를 신뢰의 근거로 삼기 좋은 정적 인프라를 보여준다.

![[Pasted image 20250817214455.png]]

- 반면에 마이크로서비스는 쉽게 수백, 수천 개의 서비스로 불어나므로 정적 환경에서는 서비스를 운영할 수 없다.
- 이런 이유로 클라우드 컴퓨팅이나 컨테이너 오케스트레이션 같은 동적 환경을 활용하는데, 여기서 서비스는 수많은 서버로 스케줄링되고 수명이 짧다.
- 따라서 **IP 주소**를 사용하는 것 같은 전통적인 방법들은 **ID의 근거로 미덥지 못하게 된다.**
- 설상가상으로 서비스가 반드시 같은 네트워크에서 실행되는 것도 아니며, 여러 클라우드 프로바이더에 걸쳐 있거나 심지어는 그림 9.2처럼 온프레미스에서도 실행 될 수 있다.
![[Pasted image 20250817214518.png]]

- 이런 문제를 해결해 고도로 동적이고 이질적인 환경에서 **ID**를 **제공**하고자 **이스티오**는 **SPIFFE** specification 사양을 사용한다.
- **SPIFFE**는 고도로 동적이고 이질적인 환경에서 **워크로드에 ID를 제공**하기 위한 일렬의 **오픈소스 표준**이다.
- **SPIFFE** 처리에 대한 더 자세한 내용과 함께 이 **SPIFFE**가 이스티오의 ID 추정을 뒷받침하는 방법을 보려면 부록 C를 참조하자.

# 9.1.5 이스티오가 SPIFFE를 구현하는 방법 How Istio implements SPIFFE - SVID

- SPIFFE ID는 [RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986) 호환 URI로, `spiffe://trust-domain/path` 형식으로 구성된다.
- 여기서는 다음과 같다.
    - trust-domain 은 개인 또는 조직 같은 ID 발급자를 나타낸다.
    - path는 trust-domain 내에서 워크로드를 유일하게 식별한다.
- path가 워크로드를 식별하는 자세한 방법은 정해져 있지 않아서 SPIFFE 명세 구현자가 결정할 수 있다.
- 이스티오에서는 이 path를 특정 워크로드가 사용하는 **서비스 어카운트**로 채운다.
- **SPIFFE ID**는 **SVID** (**S**piffe **V**erifiable **I**dentity **D**ocument, SPIFFE 검증할 수 있는 ID 문서) 라고도 하는 **X.509 인증서**로 **인코딩**되며, 이는 이스티오의 컨트롤 플레인이 워크로드마다 만들어낸다.
- 그런 다음, 이 인증서는 전송 데이터를 암호화함으로써 서비스 간 통신의 전송을 보호하는데 사용된다.
- 다시 말하지만, **부록 C**에서 이 모든 작업이 어떻게 작동하는지를 휠씬 자세히 다룬다.
- 이번 장에서는 이스티오의 기능으로 보안 태세를 개선하는 데 초점을 맞춘다.

# 들어가며 : 인증서 발급/갱신 자동화, 추가 작업(인증, 인가)
- 사이트가 프록시가 주입된 **서비스 사이의 트래픽은** **기본적**으로 **암호화**되고 서로 **인증**한다.
- **인증서를 발급하고 로테이션하는 절차를 자동화**하는 것은 매우 중요한데, 역사적으로 사람이 관리할 때 오류가 발생하기 쉬웠기 때문이다.
- 이로 인해 불필요하고 비용이 많이 드는 서비스 중단이 발생했는데, 이스티오에서 구현한 것처럼 절차를 자동화했다면 피할 수 있었을 문제였다.
- 그림 9.4는 **컨트롤 플레인에서 발급한 인증서를 사용해 서비스들이 서로 인증하고 트래픽을 암호화하는 방식**을 나타낸다.
- 이 방식을 통해 기본적으로 안전한 상태를 유지한다.
- 사실 ‘기본적으로 안전한’이라고 하면 기본적으로는 대부분 안전하다는 의미로, 메시를 더 안전하게 만들기 위해서는 아직 우리가 **수행해야 할 작업**들이 남아 있다.
![[Pasted image 20250817221537.png]]
* 워크로드는 이스티오 인증 기관에서 발급한 SVID 인증서를 사용해 서로 인증한다.

- 먼저, 서비스 메시가 **서로 인증한 트래픽만 허용하도록 설정**해야 한다.
- 왜 이것이 설치할 때 **기본값이 아닌지** 궁금할 수 있다. 이는 메시 채택을 용이하게 하려는 설계 결정이다.
- 여러 팀이 자체 서비스를 관리하는 거대 엔터프라이즈에서는 모든 서비스를 메시로 옮기기까지 몇 달 혹은 몇 년에 걸치 조직적인 노력이 필요할 수 있다.
- 두 번째로, 서비스를 인증하면 **최소 권한 원칙**을 준수할 수 있고, 각 **서비스에 정책**을 만들 수 있으며, 기능에 **필요한 최소한의 접근만 허용**할 수 있다.
- 이는 아주 중요한데, 서비스의 ID를 나타내는 인증서가 잘못된 사람에게 넘어갔을 때 피해 범위를 ID가 접근할 수 있도록 허용된 일부 서비스만으로 좁힐 수 있기 때문이다.

# (기본 정보) TLS vs mTLS

- `TLS` - [암호화방식](https://babbab2.tistory.com/4) [인증서](https://babbab2.tistory.com/5?category=960153) [Handshake](https://babbab2.tistory.com/7?category=960153)
    - TLS는 네트워크로 통신을 하는 과정에서 도청, 간섭, 위조를 방지하기 위해서 설계됨. 암호화를 통해 인증, 통신 기밀성을 제공.
    - TLS의 3단계 기본 절차: (1) 지원 가능한 알고리즘 서로 교환 (2) 키 교환, 인증 (3) 대칭키 암호로 암호화하고 메시지 인증
- `TLS` vs `MTLS` - [링크](https://www.cloudflare.com/ko-kr/learning/access-management/what-is-mutual-tls/) [소개](https://www.jacobbaek.com/1040)
    - **MTLS 절차** : **서버측**도 **클라이언트측**에 대한 **인증서**를 확인 및 **액세스 권한** 확인
![[Pasted image 20250817221650.png]]
# 9.2.1 환경 설정하기 (실습~)
- mTLS 기능 실습을 위해 3가지 서비스를 준비.
- sleep 서비스를 추가 : 레거시 워크로드로, 사이드카 프록시가 없어서 상호 인증을 할 수 없음
![[Pasted image 20250817221844.png]]
## 실습 환경 설정
```bash
# catalog와 webapp 배포
kubectl apply -f services/catalog/kubernetes/catalog.yaml -n istioinaction
kubectl apply -f services/webapp/kubernetes/webapp.yaml -n istioinaction

# webapp과 catalog의 gateway, virtualservice 설정
kubectl apply -f services/webapp/istio/webapp-catalog-gw-vs.yaml -n istioinaction

# default 네임스페이스에 sleep 앱 배포
cat ch9/sleep.yaml
...
    spec:
      serviceAccountName: sleep
      containers:
      - name: sleep
        image: governmentpaas/curl-ssl
        command: ["/bin/sleep", "3650d"]
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - mountPath: /etc/sleep/tls
          name: secret-volume
      volumes:
      - name: secret-volume
        secret:
          secretName: sleep-secret
          optional: true

kubectl apply -f ch9/sleep.yaml -n default

# 확인
kubectl get deploy,pod,sa,svc,ep
kubectl get deploy,svc -n istioinaction
kubectl get gw,vs -n istioinaction
```

## 기본 통신 확인 : 레거시 sleep 워크로드 → webapp 워크로드로 평문 요청 실행
```bash
# 요청 실행
kubectl exec deploy/sleep -c sleep -- curl -s webapp.istioinaction/api/catalog -o /dev/null -w "%{http_code}\n"

# 반복 요청
watch 'kubectl exec deploy/sleep -c sleep -- curl -s webapp.istioinaction/api/catalog -o /dev/null -w "%{http_code}\n"'
```

키알리 확인 : 네임스페이스(default, istioinaction 선택), Show Legend 클릭 후 아이콘 확인, unkonw → webapp 구간은 평문 통신

![[Pasted image 20250817222021.png]]

- 응답이 성공했다는 것은 서비스들이 올바르게 준비됐으며 webapp 서비스가 sleep 서비스의 평문 요청을 받아들였다는 사실을 보여준다.
- 기본적으로 이스티오는 평문 요청을 허용하는데, 이는 모든 워크로드를 메시로 옮길 때까지 서비스 중단을 일으키지 않고 서비스 메시를 점진적으로 채택할 수 있게 하기 위해서다.
- 그러나 PeerAuthentication 리소스로 평문 트래픽을 금지할 수 있다.



# 9.2.2 이스티오의 PeerAuthentication 리소스 이해하기*
## 들어가며
- PeerAuthentication 리소스를 사용하면 워크로드가 mTLS를 엄격하게 요구하거나 평문 트래픽을 허용하고 받아들이게 설정할 수 있다.
- 이들 각각은 **STRICT** 혹은 **PERMISSIVE** 인증 모드를 사용한다.
- 상호 mutual 인증 모드는 **다양한 범위**에서 구성할 수 있다.
    - **Mesh-wide** PeerAuthentication 정책은 **서비스 메시의 모든 워크로드**에 적용된다.
    - **Namespace-wide** PeerAuthentication 정책은 **네임스페이스 내 모든 워크로드**에 적용된다.
    - **Workload-specific** PeerAuthentication 정책은 정책에서 **명시한 셀렉터에 부합하는 모든 워크로드**에 적용된다.
## 메시 범위 정책으로 모든 미인증 트래픽 거부하기 DENYING ALL NON-AUTHENTICATED TRAFFIC USING A MESH-WIDE POLICY
- 메시의 보안을 향상시키기 위해 **STRICT** 상호 인증 모드를 **강제**하는 **메시 범위 MESH-WIDE 정책**을 만들어서 **평문 트래픽을 금지**할 수 있다.
- 메시 범위 PeerAuthentication 정책은 두 가지 조건을 충족해야 한다.
- 반드시 **이스티오를 설치한 네임스페이스**에 적용해야 하고, 이름은 ‘**default**’여야 한다.

> [!note] 
메시 범위 리소스의 이름을 ‘default’로 짓는 것은 필수가 아닌 일종의 컨벤션(convention)으로, 메시 범위 PeerAuthentication 리소스를 딱 하나만 만들기 위해서다.

```bash
#
cat ch9/meshwide-strict-peer-authn.yaml 
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default" # Mesh-wide policies must be named "default"
  namespace: "istio-system" # Istio installation namespace
spec:
  mtls:
    mode: STRICT # mutual TLS mode

# 적용
kubectl apply -f ch9/meshwide-strict-peer-authn.yaml -n istio-system

# 요청 실행
kubectl exec deploy/sleep -c sleep -- curl -s http://webapp.istioinaction/api/catalog -o /dev/null -w "%{http_code}\n"
000
command terminated with exit code 56

# 확인
kubectl get PeerAuthentication -n istio-system
kubectl logs -n istioinaction -l app=webapp -c webapp -f
kubectl logs -n istioinaction -l app=webapp -c istio-proxy -f
[2025-05-01T08:32:08.511Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.17:8080 10.10.0.16:51930 - -
[2025-05-01T08:32:10.629Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.17:8080 10.10.0.16:53366 - -
# NR → Non-Route. Envoy에서 라우팅까지 가지 못한 단계에서 발생한 에러라는 의미입니다.
# filter_chain_not_found → 해당 Listener에서 제공된 SNI(Server Name Indication), IP, 포트, ALPN 등의 조건에 맞는 filter_chain이 설정에 없다는 뜻입니다.
```

- 이는 평문 요청이 거부됐다는 것을 확인한다.
- 상호 인증 요구 사항을 STRICT로 지정하는 것은 좋은 기본값이지만, 진행 중인 프로젝트에서는 그런 급격한 변화가 실현 가능성이 없다.
- 워크로드를 옮기려면 여러 팀 간의 협업이 필요하기 때문이다.
- 더 나은 방법은 적용하는 제한을 점진적으로 늘리고, 팀들이 자신의 서비스를 서비스 메시로 옮길 수 있도록 시간을 주는 것이다.
- **PERMISSIVE** 상호 인증이 딱 그런 역할로, 워크로드가 **암호화된 요청**과 **평문 요청**을 모두 받아드릴 수 있게 허용한다.

![[Pasted image 20250817222237.png]]
![[Pasted image 20250817222248.png]]

## 상호 인증하기 않은 트래픽 허용하기 PERMITTING NON-MUTUALLY AUTHENTICATED TRAFFIC
- 네임스페이스 범위 정책을 사용하면 메시 범위 정책을 덮어 쓸 수 있고, 네임스페이스의 워크로드에 더 잘 맞는 PeerAuthentication 요구 사항을 적용할 수 있다.
- 다음 PeerAuthentication 리소스는 istioinaction 네임스페이스의 워크로드가 sleep 서비스와 같이 메시의 일부가 아닌 레거시 워크로드로부터 평문 트래픽을 받아들이도록 허용한다
```bash
cat << EOF | kubectl apply -f -
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"             # Uses the "default" naming convention so that only one namespace-wide resource exists
  namespace: "istioinaction"  # Specifies the namespace to apply the policy
spec:
  mtls:
    mode: PERMISSIVE          # PERMISSIVE allows HTTP traffic.
EOF
```

```bash
# 요청 실행
kubectl exec deploy/sleep -c sleep -- curl -s http://webapp.istioinaction/api/catalog -o /dev/null -w "%{http_code}\n"

# 확인
kubectl get PeerAuthentication -A 
NAMESPACE       NAME      MODE         AGE
istio-system    default   STRICT       2m51s
istioinaction   default   PERMISSIVE   7s

kubectl logs -n istioinaction -l app=webapp -c webapp -f
kubectl logs -n istioinaction -l app=webapp -c istio-proxy -f

# 다음 실습을 위해 삭제 : PeerAuthentication 단축어 pa
kubectl delete pa default -n istioinaction
```
- 좀 더 보안을 신경써보자.
- 미인증 트래픽은 **sleep 워크로드에서 webapp으로 향하는 것만 허용**하고, **catalog 워크로드에는 STRICT 상호 인증을 계속 유지**하자.
- 이렇게 하면 보안이 뚫렸을 때 공격 표면을 더 좁힐 수 있다.

## 워크로드별 PeerAuthentication 정책 적용하기 APPLYING WORKLOAD-SPECIFIC PEERAUTHENTICATION POLICIES
- webapp 만 목표로 하기 위해 워크로드 셀렉터를 지정해 상술했던 PeerAuthentication 정책을 업데이트 하자.
- 이로써 셀렉터에 부합하는 워크로드에만 적용될 것이다.
- 또한 이름을 ‘default’에서 webapp으로 바뀌자.
- 동작이 바꾸지는 않지만, 네임스페이스 전체에 적용되는 PeerAuthentication 정책만 ‘default’로 짓는 컨벤션을 따르려는 것이다.
```bash
# istiod 는 PeerAuthentication 리소스 생성을 수신하고, 이 리소스를 엔보이용 설정으로 변환하며, 
# LDS(Listener Discovery Service)를 사용해 서비스 프록시에 적용
docker exec -it myk8s-control-plane istioctl proxy-status
kubectl logs -n istio-system -l app=istiod -f
...
2025-05-01T09:48:32.854911Z     info    ads     LDS: PUSH for node:catalog-6cf4b97d-2r9bn.istioinaction resources:23 size:85.4kB
2025-05-01T09:48:32.855510Z     info    ads     LDS: PUSH for node:webapp-7685bcb84-jcg7d.istioinaction resources:23 size:94.0kB
...

#
cat ch9/workload-permissive-peer-authn.yaml
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "webapp"
  namespace: "istioinaction"
spec:
  selector:
    matchLabels:
      app: "webapp"  # 레이블이 일치하는 워크로드만 PERMISSIVE로 동작
  mtls:
    mode: PERMISSIVE

kubectl apply -f ch9/workload-permissive-peer-authn.yaml
kubectl get pa -A

# 요청 실행
kubectl logs -n istioinaction -l app=webapp -c webapp -f
kubectl logs -n istioinaction -l app=webapp -c istio-proxy -f
kubectl exec deploy/sleep -c sleep -- curl -s http://webapp.istioinaction/api/catalog -o /dev/null -w "%{http_code}\n"

#
kubectl logs -n istioinaction -l app=catalog -c catalog -f
kubectl logs -n istioinaction -l app=catalog -c istio-proxy -f
kubectl exec deploy/sleep -c sleep -- curl -s http://catalog.istioinaction/api/items -o /dev/null -w "%{http_code}\n"
2025-05-01T09:32:00.197Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.18:3000 10.10.0.16:33192 - -
...
```

- 성공 응답을 반환한다! 메시 범위 정책으로 엄격한 기본값을 적용했다.
- 그러나 일부 서비스(뒤처진 것들)에는 그 서비스들이 메시로 옮겨질 때까지 상호 인증이 아닌 트래픽도 허용되도록 워크로드별 정책을 사용한다.

![[Pasted image 20250817222615.png]]
>[!note] istiod 는 PeerAuthentication 리소스 생성을 수신하고, 이 리소스를 엔보이용 설정으로 변환하며, LDS(Listener Discovery Service)를 사용해 서비스 프록시에 적용한다. 구성된 정책들은 들어오는 요청마다 평가된다. The configured policies are evaluated for every incoming request.

## 두 가지 추가적인 상호 인증 모드 TWO ADDITIONAL MUTUAL AUTHENTICATION MODES

- 대부분의 경우 **STRICT** 나 **PERMISSIVE** 모드를 사용할 것이다. 그러나 두 가지 모드가 더 있다.
    - **UNSET** : 부모의 PeerAuthentication 정책을 상속한다. _Inherit the PeerAuthentication policy of the parent._
    - **DISABLE** : 트래픽을 터널링하지 않는다. 그냥 보낸다. _Do not tunnel the traffic; send it directly to the service._
- PeerAuthentication 리소스를 이렇게 사용할 수 있다.
- 상호 인증 트래픽, 평문 트래픽 등 워크로드로 터널링할 트래픽 유형을 지정하거나, 요청을 프록시로 보내지 않고 애플리케이션으로 바로 포워딩할 수 있다.
- 다음 절에서는 상호 TLS를 사용할 때 트래픽이 암호화되는지 확인해보자.

## tcpdump로 서비스 간 트래픽 스니핑하기 EAVESDROPPING ON SERVICE-TO-SERVICE TRAFFIC USING TCPDUMP
- 이스티오 프록시에는 **tcpdump**가 **설치**돼 있다. 이 도구는 네트워크 인터페이스르 통과하는 네트워크 트래픽을 포착하고 분석한다.
- tcpdump 는 보안 때문에 권한 **privileged** permission 이 필요한데, 기본적으로 이 권한은 꺼져 있다.
- 이 권한을 켜려면 istioctl로 속성 `values.global.proxy.privileged=true` 로 설정해 이스티오 설치를 업데이트하자.
>[!note]
>격상시킨 서비스 프록시의 권한은 악의적 공격의 매개체가 될 수 있다. 운영 환경 클러스터에서 이스티오를 설치할 때는 프록시의 권한을 격상시키지 말자. 서비스 하나를 빠르게 디버깅하고 싶을때는 kubectl edit로 디플로이먼트의 필드를 수작업으로 바꿀 수 있다.

```bash
# demo 프로파일 컨트롤 플레인 배포 시 istio-proxy 에 privileged 설정 >> 이미 설정 되어 있음
istioctl install --set profile=demo --set values.global.proxy.privileged=true -y
```

```bash
# 확인
kubectl get istiooperator -n istio-system installed-state -o yaml
...
      proxy:
        ...
        privileged: true
...

kubectl get pod -n istioinaction -l app=webapp -o json
                        "image": "docker.io/istio/proxyv2:1.17.8",
                        "imagePullPolicy": "IfNotPresent",
                        "name": "istio-proxy",
                        ...
                        "securityContext": {
                            "allowPrivilegeEscalation": true,
                            "capabilities": {
                                "drop": [
                                    "ALL"
                                ]
                            },
                            "privileged": true,
                            "readOnlyRootFilesystem": true,
                            "runAsGroup": 1337,
                            "runAsNonRoot": true,
                            "runAsUser": 1337
                        },
...

# 
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- whoami
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- id
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- sudo whoami
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- sudo tcpdump -h

```

## 파드 트래픽을 스니핑 sniffing 해보자
```bash
# 패킷 모니터링 실행 해두기
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy \
  -- sudo tcpdump -l --immediate-mode -vv -s 0 '(((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0) and not (port 53)'
# -l : 표준 출력(stdout)을 라인 버퍼 모드로 설정. 터미널에서 실시간으로 결과를 보기 좋게 함 (pipe로 넘길 때도 유용).
# --immediate-mode : 커널 버퍼에서 패킷을 모아서 내보내지 않고, 캡처 즉시 사용자 공간으로 넘김 → 딜레이 최소화.
# -vv : verbose 출력. 패킷에 대한 최대한의 상세 정보를 보여줌.
# -s 0 : snap length를 0으로 설정 → 패킷 전체 내용을 캡처. (기본값은 262144 bytes, 예전 버전에서는 68 bytes로 잘렸음)
# '(((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0) and not (port 53)' : DNS패킷 제외하고 TCP payload 길이가 0이 아닌 패킷만 캡처
# 즉, SYN/ACK/FIN 같은 handshake 패킷(데이터 없는 패킷) 무시, 실제 데이터 있는 패킷만 캡처
# 결론 : 지연 없이, 전체 패킷 내용을, 매우 자세히 출력하고, DNS패킷 제외하고 TCP 데이터(payload)가 1 byte 이상 있는 패킷만 캡처

# 요청 실행
kubectl exec deploy/sleep -c sleep -- curl -s webapp.istioinaction/api/catalog -o /dev/null -w "%{http_code}\n"
...
## (1) sleep -> webapp 호출 HTTP
14:07:24.926390 IP (tos 0x0, ttl 63, id 63531, offset 0, flags [DF], proto TCP (6), length 146)
    10-10-0-16.sleep.default.svc.cluster.local.32828 > webapp-7685bcb84-hp2kl.http-alt: Flags [P.], cksum 0x14bc (incorrect -> 0xa83b), seq 2741788650:2741788744, ack 3116297176, win 512, options [nop,nop,TS val 490217013 ecr 2804101520], length 94: HTTP, length: 94
	GET /api/catalog HTTP/1.1
	Host: webapp.istioinaction
	User-Agent: curl/8.5.0
	Accept: */*

## (2) webapp -> catalog 호출 HTTPS
14:07:24.931647 IP (tos 0x0, ttl 64, id 18925, offset 0, flags [DF], proto TCP (6), length 1304)
    webapp-7685bcb84-hp2kl.37882 > 10-10-0-19.catalog.istioinaction.svc.cluster.local.3000: Flags [P.], cksum 0x1945 (incorrect -> 0x9667), seq 2146266072:2146267324, ack 260381029, win 871, options [nop,nop,TS val 1103915113 ecr 4058175976], length 1252

## (3) catalog -> webapp 응답 HTTPS
14:07:24.944769 IP (tos 0x0, ttl 63, id 7029, offset 0, flags [DF], proto TCP (6), length 1789)
    10-10-0-19.catalog.istioinaction.svc.cluster.local.3000 > webapp-7685bcb84-hp2kl.37882: Flags [P.], cksum 0x1b2a (incorrect -> 0x2b6f), seq 1:1738, ack 1252, win 729, options [nop,nop,TS val 4058610491 ecr 1103915113], length 1737

## (4) webapp -> sleep 응답 HTTP
14:07:24.946168 IP (tos 0x0, ttl 64, id 13699, offset 0, flags [DF], proto TCP (6), length 663)
    webapp-7685bcb84-hp2kl.http-alt > 10-10-0-16.sleep.default.svc.cluster.local.32828: Flags [P.], cksum 0x16c1 (incorrect -> 0x37d1), seq 1:612, ack 94, win 512, options [nop,nop,TS val 2804101540 ecr 490217013], length 611: HTTP, length: 611
	HTTP/1.1 200 OK
	content-length: 357
	content-type: application/json; charset=utf-8
	date: Thu, 01 May 2025 14:07:24 GMT
	x-envoy-upstream-service-time: 18
	server: istio-envoy
	x-envoy-decorator-operation: webapp.istioinaction.svc.cluster.local:80/*

	[{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00"},{"id":2,"color":"cyan","department":"Clothing","name":"Atlas Shirt","price":"127.00"},{"id":3,"color":"teal","department":"Clothing","name":"Small Metal Shoes","price":"232.00"},{"id":4,"color":"red","department":"Watches","name":"Red Dragon Watch","price":"232.00"}] [|http]
...
```

```bash
#
kubectl get svc,ep -n istioinaction
NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/catalog   ClusterIP   10.200.1.46    <none>        80/TCP    7h4m
service/webapp    ClusterIP   10.200.1.201   <none>        80/TCP    7h4m

NAME                ENDPOINTS         AGE
endpoints/catalog   10.10.0.19:3000   7h4m
endpoints/webapp    10.10.0.20:8080   7h4m

#
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy \
  -- sudo tcpdump -l --immediate-mode -vv -s 0 'tcp port 3000 or tcp port 8080'

# 요청 실행
kubectl exec deploy/sleep -c sleep -- curl -s webapp.istioinaction/api/catalog -o /dev/null -w "%{http_code}\n"
...
```
## 워크로드 ID가 워크로드 서비스 어카운트에 연결돼 있는지 확인하기 VERIFYING THAT WORKLOAD IDENTITIES ARE TIED TO THE WORKLOAD SERVICE ACCOUNT
- 상호 인증을 다룬 절을 끝내기 전에 발급된 인증서가 유효한 SVID 문서인지, SPIFFE ID가 인코딩돼 있는지, 그 ID가 워크로드 서비스 어카운트와 일치하는지 확인해보자.
- openssl 명령어를 사용해 catalog 워크로드의 X.509 인증서 내용물을 확인한다.

```bash
# (참고) 패킷 모니터링 : 아래 openssl 실행 시 동작 확인
kubectl exec -it -n istioinaction deploy/catalog -c istio-proxy \
  -- sudo tcpdump -l --immediate-mode -vv -s 0 'tcp port 3000'
  

# catalog 의 X.509 인증서 내용 확인
kubectl -n istioinaction exec deploy/webapp -c istio-proxy -- ls -l /var/run/secrets/istio/root-cert.pem
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- openssl x509 -in /var/run/secrets/istio/root-cert.pem -text -noout
...

kubectl -n istioinaction exec deploy/webapp -c istio-proxy -- openssl -h
kubectl -n istioinaction exec deploy/webapp -c istio-proxy -- openssl s_client -h

# openssl s_client → TLS 서버에 연결해 handshake와 인증서 체인을 보여줌
# -showcerts → 서버가 보낸 전체 인증서 체인 출력
# -connect catalog.istioinaction.svc.cluster.local:80 → Istio 서비스 catalog로 TCP 80 연결
# -CAfile /var/run/secrets/istio/root-cert.pem → Istio의 root CA로 서버 인증서 검증
# 결론 : Envoy proxy에서 catalog 서비스로 연결하여 TLS handshake 및 인증서 체인 출력 후 사람이 읽을 수 있는 형식으로 해석
kubectl -n istioinaction exec deploy/webapp -c istio-proxy \
  -- openssl s_client -showcerts \
  -connect catalog.istioinaction.svc.cluster.local:80 \
  -CAfile /var/run/secrets/istio/root-cert.pem | \
  openssl x509 -in /dev/stdin -text -noout
...
        Validity 
            Not Before: May  1 09:55:10 2025 GMT # 유효기간 1일 2분
            Not After : May  2 09:57:10 2025 GMT
        ...
        X509v3 extensions:
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication # 사용처 : 웹서버, 웹클라이언트
            ...
            X509v3 Subject Alternative Name: critical
                URI:spiffe://cluster.local/ns/istioinaction/sa/catalog # SPIFFE ID 확인

# catalog 파드의 서비스 어카운트 확인
kubectl describe pod -n istioinaction -l app=catalog | grep 'Service Account'
Service Account:  catalog
```

- 루트 인증서 서명 확인
    - `openssl verify` 로 인증 기관 CA 루트 인증서에 대해 서명을 확인함으로써 X.509 SVID의 내용물이 유효한지 살펴보자.
    - 루트 인증서는 istio-proxy 컨테이너에서 `/var/run/secrets/istio/root-cert.pem` 경로에 마운트돼 있다.
```bash
# webapp.istio-proxy 쉘 접속
kubectl -n istioinaction exec -it deploy/webapp -c istio-proxy -- /bin/bash
-----------------------------------------------
# 인증서 검증
openssl verify -CAfile /var/run/secrets/istio/root-cert.pem \
  <(openssl s_client -connect \
  catalog.istioinaction.svc.cluster.local:80 -showcerts 2>/dev/null)
/dev/fd/63: OK
# 검증에 성공 시 OK 메시지 출력: 이스티오 CA가 인증서에 서명했으며, 내부 데이터가 믿을 수 있다는 것임을 알려줌.

exit
-----------------------------------------------
```

- 이제 참가자 간 peer-to-peer 인증을 용의하게 하는 모든 구성 요소를 검증했으므로, **발급된 ID는 검증**할 수 있는 것이고 **트래픽은 안전하다는 것을 확신**할 수 있다.
- 검증할 수 있는 ID가 접**근 제어의 선행 조건**이다. 다시 말해, 워크로드의 ID를 알고 있으므로 수행할 수 있는 작업을 정의할 수 있다.
- 다음 절에서는 **인가 정책**을 살펴본다.

# C.1 PKI를 사용한 인증 Authentication using PKI (public key infrastructure)
- 들어가며
    - World Wide Web에서 통신 당사자는 **PKI** **P**ublic **K**ey **I**nfrastructure (공개 키 인프라) 규격을 따라 발급한 디지털 서명 인증서를 사용해 인증한다.
    - PKI는 **절차를 정의**하는 프레임워크인데, 이 절차는 서버(웹 앱 등)에는 자신의 정체를 증명할 수 있는 **디지털 인증서**를 제공하고 클라이언트에는 디지털 **인증서의 유효성을 검증**할 수 있는 수단을 제공한다. [https://www.securew2.com/blog/public-key-infrastructure-explained](https://www.securew2.com/blog/public-key-infrastructure-explained)
    - PKI에서 제공하는 인증서에는 **공개 키**와 **개인 키**가 있다. 클라이언트에게 인증서를 인증 수단으로 제시하는데, 공개 키는 이 인증서 안에 포함된다.
    - 클라이언트는 공개된 네트워크에서 서버로 데이터를 전송하기 전에 공개 키를 사용해 데이터를 암호화하며, 개인 키를 가진 서버만이 데이터를 복호화할 수 있다.        
    - 이런 방식으로 데이터는 전송 중에 안전하게 보호된다.

>[!note] 
>공개 키 인증서의 표준 형식을 X.509 인증서라고 한다. 이 책에서는 X.509 인증서라는 용어와 디지털 인증서라는 용어를 같은 뜻으로 사용한다.

* 국제 인터넷 표준화 기구 IETF는 전송 계층 보안 TLS Transport Layer Security 프로토콜(PKI를 사용하기는 하지만 PKI만 사용해야 하는 것은 아님)을 정의하고, X.509 인증서를 공급해 트래픽 인증 및 암호화를 용의하게 했다.

## C.1.1 TLS 및 최종 사용자 인증을 통한 트래픽 암호화 Traffic encryption via TLS and end-user authentication

TLS 프로토콜은 TLS 핸드셰이크 절차에서 서버의 유효성을 인증하고 트래픽 대칭 키 암호화용 키를 안전하게 교환하는 데 X.509 인증서를 기본 메커니즘으로 사용한다. (그림 C.1 참조)
![[Pasted image 20250817224110.png]]
1. **클라이언트**가 자신이 지원하는 TLS 버전과 암호화 수단을 포함한 `ClientHello` 로 핸드셰이크를 시작한다.
2. **서버**는 `ServerHello` 와 **자신의 X.509 인증서**로 응답한다. 인증서에는 **서버의 ID 정보**와 **공개 키**가 포함돼 있다.
3. **클라이언트**는 서버의 인증서 데이터가 **변조**되지 않았음을 확인하고 **신뢰 체인**을 검증한다.
4. 검증에 성공하면, **클라이언트**는 **서버**에 **비밀 키**를 보낸다. 이 키는 **임의**로 생성한 **문자열**을 **서버**의 **공개 키**로 **암호화**한 것이다.
5. **서버**는 자신의 개인 키로 비밀 키를 **복호화**하고, 복호화된 비밀 키로 ‘**finished**’ 메시지를 **암호화**해 **클라이언트**로 보낸다.
6. **클라이언트**도 **비밀 키**로 암호화한 ‘**finished**’ 메시지를 **서버**에 보내면 **TLS 핸드셰이크**가 완료된다.

- **TLS 핸드셰이크**의 결실은 **클라이언트가 서버를 인증**했고 **대칭 키를 안전하게 교환**했다는 것이다.
- 이 대칭 키는 이 커넥션에서 **클라이언트와 서버를 오가는 트래픽을 암호화**하는데 사용한다.
- 이런 방식이 비대칭 암호화보다 성능이 더 좋기 때문이다.
- 최종 사용자에게 이런 절차는 브라우저가 투명하게 수행하는 것으로, 주소 표시줄에 녹색 자물쇠로 표시돼 수신자가 인증됐고 트래픽이 암호화돼 수신자만 복호화할 수 있다는 것을 확인해준다.
- **서버**에서 **최종 사용자를 인증**하는 것은 구현하기 나름이다.
- 여러 가지 방법이 있지만, 그 모든 방법의 핵심은 비밀번호를 알고 있는 사용자가 **세션 쿠키**나 **JWT**(JSON Web Token)를 받는 것이다.
- 이때 **JWT**는 **수명이 짧고** 사용자의 후속 요청을 서버에 인증하기 위한 정보를 포함하는 것이 이상적이다.
- 이스티오는 JWT를 사용하는 최종 사용자 인증을 지원한다.
- 실제로 동작하는 모습은 9.4절에서 살펴봤다.

# C.2 SPIFFE: 모든 이를 위한 안전한 운영 환경 ID 프레임워크 Secure Production Identity Framework for Everyone (실습)
- 들어가며
    - SPIFFE는 고도로 동적이며 이질적인 환경에서 워크로드에 ID를 제공하기 위한 오픈소스 표준 집합이다.
        
    - ID를 발급하고 부트스트랩하기 위해 SPIFFE는 다음 사양을 정의한다.
        
        - **SPIFFE ID** : 신뢰 도메인 내에서 서비스를 고유하게 구별한다.
        - **Workload Endpoint** : 워크로드의 ID를 부트스트랩한다.
        - **Workload API** : SPIFFE ID가 포함된 인증서를 서명하고 발급한다.
        - **SVID** **S**PIFFE **V**erifiable **I**dentity **D**ocument : 워크로드 API가 발급한 인증서로 표현된다.
    - SPIFFE 사양은 SPIFFE ID 형식으로 워크로드에 ID를 발급하고 이를 SVID에 인코딩하는 절차를 정의할 뿐 아니라, 컨트롤 플레인 구성 요소(워크로드 API)와 데이터 플레인 구성 요소(워크로드 엔드포인트)가 워크로드의 ID를 검증하고 할당하고 형식의 유효성을 검사하기 위해 협동하는 방법도 정의한다.
        
    - 이스티오가 이런 사양을 구현하므로 이에 대한 더 깊은 이해가 필요하다.

## C.2.1 SPIFFE ID: Workload identity 워크로드 ID
- SPIFFE ID는 RFC 3986 호환 URI로, `spiffe://trust-domain/path` 형식을 따른다.
    - trust-domain 은 개인이나 조직 같은 ID 발급자를 나타낸다.
    - path는 trust-domain 내에서 워크로드를 고유하게 식별한다.
- **경로** path로 워크로드를 식별하는 방법의 세부 사항에는 제약이 없으며 SPIFFE 사양 **구현자가 결정**할 수 있다.
- 이 부록에서는 이스티오가 **쿠버네티스 서비스 어카운트**를 사용해 워크로드를 식별하는 **경로**를 정의하는 방법을 살펴본다.

## C.2.2 Workload API 워크로드 API
- **워크로드 API**는 SPIFFE 사양에서 **컨트롤 플레인** 구성 요소를 나타내며, 워크로드가 자신의 ID를 정의하는 SVID 형식 디지털 인증서를 가져갈 수 있도록 엔드포인트를 노출한다.
- **워크로드 API**는 두 가지 주요 기능은 다음과 같다.
    - 워크로드가 제출한 **인증서 서명 요청** CSR에 인증 기관 CA 개인 키로 서명함으로써 워크로드에 인증서 발급
    - 워크로드 엔드포인트에서 해당 기능을 사용할 수 있도록 API 노출
- 사양 specification sets 은 워크로드가 **자신의 ID를 정의하는 비밀이나 기타 정보를 보유해서는 안 된다**는 **제한**(규칙)을 둔다.
- 그렇지 않으면, 해당 비밀에 접근할 수 있는 악의적인 사용자가 시스템을 쉽게 악용할 수 있기 때문이다.
- 이 제한 때문에 워크로드에는 인증 수단이 없어 워크로드 API로 보안 통신을 시작할 수 없다.
- 이 상황을 해결하기 위해 SPIFFE는 **워크로드 엔드포인트 사양**을 정의한다.
- 이 사양은 **데이터 플레인** 구성 요소를 나타내고, **워크로드의 ID를 부트스트랩**하는 데 필요한 모든 작업을 수행한다.
- 예를 들어, **워크로드 API와 보안 통신을 시작**하거나 도청 또는 중간자 **공격에 취약하지 않게** **SVID를 가져오**는 등의 활동을 수행한다.
## C.2.3 Workload endpoints 워크로드 엔드포인트
- 워크로드 엔드포인트는 SPIFFE 사양의 데이터 플레인 구성 요소를 나타낸다. 이는 모든 워크로드와 함께 배포돼 다음 기능을 제공한다.
    - **워크로드 증명** attestation
        - 커널 검사 kernel introspection 또는 orchestrator interrogation (쿼리, 질문) 같은 방법을 사용해 워크로드의 ID를 확인한다.
    - **워크로드 API 노출** exposure
        - 워크로드 API와 보안 통신을 시작하고 유지한다. 이 보안 통신은 SVID를 가져오고 로테이션하는 데 사용한다.
- 그림 C.2는 **워크로드에 ID를 발급하는 단계**의 개요를 보여준다.

![[Pasted image 20250817224426.png]]

1. **워크로드 엔드포인트**는 워크로드의 **무결성을** 확인하고(즉, 워크로드 증명을 수행하고) **SPIFFIE ID가 인코딩된 CSR을 생성**한다.
2. 워크로드 엔드포인트는 **서명을 위해 워크로드 API에 CSR을 제출**한다.
3. **워크로드 API는 CSR을 서명하고 디지털 서명된 인증서로 응답**한다.
    - 이 인증서의 **SAN**의 URI 확장에는 SPIFFE ID가 있다.
    - 이 인증서는 워크로드 ID를 나타내는 **SVID**이다.

## C.2.4 SPIFFE Verifiable Identity Documents 검증할 수 있는 ID 문서

- **SVID** (SPIFFE 검증할 수 있는 ID 문서) 는 **워크로드의 정체**를 나타내는 **검증**할 수 있는 **문서**다.
- 검증할 수 있다는 것이 가장 중요한 속성인데, 그렇지 않으면 수신자가 워크로드의 정체를 신뢰할 수 없기 때문이다.
- 사양은 SVID 표현 기준을 충족하는 문서로 두 가지 유형인 **X.509 인증서**와 **JWT**를 정의한다.
- 둘 다 다음과 같은 요소로 구성된다.
    - SPIFFE ID, 워크로드 ID를 나타낸다.
    - 유효한 서명, SPIFFE ID가 변조되지 않았음을 확인한다.
    - (선택 사항) 워크로드 간에 보안 통신 채널을 구축하기 위한 공개 키
- **이스티오는 SVID**를 **X.509 인증서**로 구현한다.
- 그 방법은 **SAN** **S**ubject **A**lternative **N**ame 확장에 SPIFFE ID를 URI로 인코딩하는 것이다.
- X.509 인증서를 사용하면 추가적인 이점이 있는데, 워크로드가 서로 간의 트래픽을 **상호 인증**하고 **암호화**할 수 있다는 것이다. (그림 C.3 참조)
![[Pasted image 20250817224553.png]]

- 이스티오가 SPIFFE 사양을 구현함으로써, 모든 워크로드가 각자의 ID를 공급받고 그 ID를 증거로 인증서를 받는다는 것이 자동으로 보장된다.
- 이런 인증서는 **상호 인증**과 모든 서비스 간 **통신을 암호화**하는 데 사용한다.
- 그러므로 이 기능을 **자동 상호 TLS**라고 한다. _Hence this feature is called **auto mTLS**._
## C.2.5 How Istio implements SPIFFE 이스티오가 SPIFFE를 구현하는 방법
- 이스티오를 사용하면 다음 두 구성 요소가 협업해 워크로드에 ID를 제공한다.
    - **ID를 부트스트랩하는 워크로드 엔드포인트** _(데이터플레인, 이스티오 프록시 pilot agent)_
    - **인증서를 발급하는 워크로드 API** _(컨트롤플레인, istiod 의 Istio CA)_
- 이스티오에서 **워크로드 엔드포인트** 사양은 워크로드와 함께 배포되는 **이스티오 프록시가 구현**한다.
- 이스티오 프록시는 **ID를 부트스트랩**하고 **이스티오 CA에서 인증서를 가져**오는데, 이스티오 CA는 **istiod**의 구성 요소로 **워크로드 API** 사양을 **구현**한다.
- 그림 C.4는 이스티오가 SPIFFE 구성 요소를 구현하는 방법을 보여준다.
![[Pasted image 20250817224633.png]]
- **워크로드 엔드포인트**는 ID 부트스트랩을 수행하는 **이스티오 파일럿 에이전**트로 구현한다.
- **워크로드 API**는 인증서를 발급하는 **이스티오 CA**로 구현한다.
- 이스티오에서 ID를 발급하는 워크로드는 서비스 프록시다.
    
- 이는 이스티오가 SPIFFE를 구현하는 방법을 고수준에서 살펴본 것이다.
- 해당 내용을 이해하고 기억하기 위해 이 과정을 단계별로 살펴보자.
## C.2.6 Step-by-step bootstrapping of workload identity* 워크로드 ID의 단계별 부트스트랩
- 기본적으로 쿠버네티스에서 초기화된 모든 파드에는 `/var/run/secrets/kubernetes.io/serviceaccount/` 경로에 시크릿이 마운트돼 있다.
- 이 시크릿에는 쿠버네티스 API 서버와 안전하게 통신하는 데 필요한 모든 데이터가 포함돼 있다.
    - ca.crt 는 쿠버네티스 API 서버가 발급한 인증서의 유효성을 검증한다.
    - 네임스페이스는 파드가 위치한 곳을 나타낸다.
    - 서비스 어카운트 토큰에는 파드를 나타내는 서비스 어카운트에 대한 (토큰)클레임들이 포함된다.
        - _The service account token contains a set of claims for the service account representing the Pod._
- ID 부트스트랩 과정에서 가장 중요한 요소는 쿠버네티스 API가 발급한 토큰이다.
- 토큰의 페이로드는 수정할 수 없는데, 수정하면 서명 유효성 검사를 통과하지 못하기 때문이다.
- 페이로드에는 애플리케이션을 식별하는 데이터가 포함된다.
```bash
#
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- ls -l /var/run/secrets/kubernetes.io/serviceaccount/
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
TOKEN=$(kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# 헤더 디코딩 
echo $TOKEN | cut -d '.' -f1 | base64 --decode | sed 's/$/}/' | jq
{
  "alg": "RS256",
  "kid": "nKgUYnbjH9BmgEXYbu56GFoBxwDF_jF9Q6obIWvinAM"
}

# 페이로드 디코딩
echo $TOKEN | cut -d '.' -f2 | base64 --decode | sed 's/$/}/' | jq
{
  "aud": [
    "https://kubernetes.default.svc.cluster.local"
  ],
  "exp": 1777689454,
  "iat": 1746153454,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "kubernetes.io": {
    "namespace": "istioinaction",
    "pod": {
      "name": "webapp-7685bcb84-hp2kl",
      "uid": "98444761-1f47-45ad-b739-da1b7b22013a"
    },
    "serviceaccount": {
      "name": "webapp",
      "uid": "5a27b23e-9ed6-46f7-bde0-a4e4684949c2"
    },
    "warnafter": 1746157061
  },
  "nbf": 1746153454,
  "sub": "system:serviceaccount:istioinaction:webapp"
}

# (옵션) brew install jwt-cli  # Linux 툴 추천 부탁드립니다.
jwt decode $TOKEN
Token header
------------
{
  "alg": "RS256",
  "kid": "nKgUYnbjH9BmgEXYbu56GFoBxwDF_jF9Q6obIWvinAM"
}

Token claims
------------
{
  "aud": [ # 이 토큰의 대상(Audience) : 토큰이 어떤 API나 서비스에서 사용될 수 있는지 정의 -> k8s api가 aud 가 일치하는지 검사하여 올바른 토큰인지 판단.
    "https://kubernetes.default.svc.cluster.local"
  ],
  "exp": 1777689454, # 토큰 만료 시간 Expiration Time (Unix timestamp, 초 단위) , date -r 1777689454 => (1년) Sat May  2 11:37:34 KST 2026
  "iat": 1746153454, # 토큰 발급 시간 Issued At (Unix timestamp), date -r 1746153454 => Fri May  2 11:37:34 KST 2025
  "iss": "https://kubernetes.default.svc.cluster.local", # Issuer, 토큰을 발급한 주체, k8s api가 발급
  "kubernetes.io": {
    "namespace": "istioinaction",
    "pod": {
      "name": "webapp-7685bcb84-hp2kl",
      "uid": "98444761-1f47-45ad-b739-da1b7b22013a" # 파드 고유 식별자
    },
    "serviceaccount": {
      "name": "webapp",
      "uid": "5a27b23e-9ed6-46f7-bde0-a4e4684949c2" # 서비스 어카운트 고유 식별자
    },
    "warnafter": 1746157061 # 이 시간 이후에는 새로운 토큰을 요청하라는 Kubernetes의 신호 (토큰 자동 갱신용) date -r 1746157061 (1시간) => Fri May  2 12:37:41 KST 2025
  },
  "nbf": 1746153454, # Not Before, 이 시간 이전에는 토큰이 유효하지 않음. 보통 iat와 동일하게 설정됩니다.
  "sub": "system:serviceaccount:istioinaction:webapp" # 토큰의 주체(Subject)
}

# sa 에 토큰 유효 시간 3600초 = 1시간 + 7초
kubectl get pod -n istioinaction -l app=webapp -o yaml
...
    - name: kube-api-access-nt4qb
      projected:
        defaultMode: 420
        sources:
        - serviceAccountToken:
            expirationSeconds: 3607
            path: token
...
```

- 파일럿 에이전트는 토큰을 디코딩하고 이 페이로드 데이터를 사용해 SPIFFE ID(예 `spiffe://cluster.local/ns/istioinaction/sa/default`)를 생성한다.
- 이 SPIFFE ID는 CSR안에서 URI 유형의 SAN 확장으로 사용한다.
- 이스티오 CA로 보낸 **요청**에 **토큰과 CSR이 모두 전송**되며, CSR에 대한 **응답으로 발급된 인증서가 반환**된다.
    - _Both the **token** and the **CSR** are sent in the request to the Istio CA to get a certificate issued for the CSR._
- CSR에 서명하기 전에 **이스티오 CA**는 **TokenReview API**를 사용해 토큰이 쿠버네티스 API가 발급한 것이 맞는지 확인한다.
- 이는 SPIFFE 사양에서 약간 벗어난 것인데, SPIFFE 사양에서는 워크로드 엔드포인트(이스티오 에이전트)가 워크로드 증명을 수행해야 하기 때문이다.
- 검증을 통과하면 CSR에 서명하고, 결과 인증서가 파일럿 에이전트에 반환된다.
```bash
#
kubectl api-resources | grep -i token
tokenreviews                                   authentication.k8s.io/v1               false        TokenReview

kubectl explain tokenreviews.authentication.k8s.io
...
DESCRIPTION:
     TokenReview attempts to authenticate a token to a known user. Note:
     TokenReview requests may be cached by the webhook token authenticator
     plugin in the kube-apiserver.
...

# Kubernetes API 서버에 TokenReview API 를 호출하여 토큰이 여전히 유효한지 확인 : C(Create)
## 이때 사용되는 Kubernetes API 가 POST /apis/authentication.k8s.io/v1/tokenreviews
## 즉, istiod가 이 API를 호출하려면 tokenreviews.authentication.k8s.io 리소스에 create 권한이 필요. C(Create)
kubectl rolesum istiod -n istio-system
...
• [CRB] */istiod-clusterrole-istio-system ⟶  [CR] */istiod-clusterrole-istio-system
  Resource                                                                                                                                             Name               Exclude    Verbs    G L W C U P D DC  
  ...
  signers.certificates.k8s.io                                                                                                             [kubernetes.io/legacy-unknown]    [-]    [approve]  ✖ ✖ ✖ ✖ ✖ ✖ ✖ ✖   
  subjectaccessreviews.authorization.k8s.io                                                                                                            [*]                  [-]       [-]     ✖ ✖ ✖ ✔ ✖ ✖ ✖ ✖   
  tokenreviews.authentication.k8s.io                                                                                                                   [*]                  [-]       [-]     ✖ ✖ ✖ ✔ ✖ ✖ ✖ ✖   
  validatingwebhookconfigurations.admissionregistration.k8s.io                                                                                         [*]                  [-]       [-]     ✔ ✔ ✔ ✖ ✔ ✖ ✖ ✖   
...
```

>[!note]
>istiod 컨테이너 내부에서 자신의 token 으로 k8s 에 tokenreviews api 호출해보는 실습 명령어 추가해두기. (https 로 호출해야되고, k8s root ca 인증서를 지정 필요)

>[!note]
>istio-proxy 배포 실행 로그와 위 과정 실행 절차를 확인하는 실습 명령어 추가해두기.

파일럿 에이전트는 SDS Secrets Discovery Service 를 통해 인증서와 키를 엔보이 프록시로 전달하고, 이로써 ID 부트스트랩 과정이 마무리된다.
![[Pasted image 20250817224854.png]]

```bash
# 유닉스 도메인 소켓 listen 정보 확인
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- ss -xpl
Netid           State            Recv-Q           Send-Q                                                    Local Address:Port                      Peer Address:Port           Process                                        
u_str           LISTEN           0                4096                                                etc/istio/proxy/XDS 13207                                * 0               users:(("pilot-agent",pid=1,fd=11))           
u_str           LISTEN           0                4096                       ./var/run/secrets/workload-spiffe-uds/socket 13206                                * 0               users:(("pilot-agent",pid=1,fd=10))  

kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- ss -xp
Netid State Recv-Q Send-Q                                Local Address:Port    Peer Address:Port   Process                             
u_str ESTAB 0      0      ./var/run/secrets/workload-spiffe-uds/socket 21902              * 23737   users:(("pilot-agent",pid=1,fd=16))
u_str ESTAB 0      0                               etc/istio/proxy/XDS 1079087            * 1080955 users:(("pilot-agent",pid=1,fd=8)) 
...

# 유닉스 도메인 소켓 정보 확인
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- lsof -U
COMMAND   PID        USER   FD   TYPE             DEVICE SIZE/OFF    NODE NAME
pilot-age   1 istio-proxy    8u  unix 0x00000000bda7185a      0t0 1079087 etc/istio/proxy/XDS type=STREAM # 소켓 경로 및 스트림 타입
pilot-age   1 istio-proxy   10u  unix 0x0000000009112f4b      0t0   13206 ./var/run/secrets/workload-spiffe-uds/socket type=STREAM # SPIFFE UDS (SPIFFE SVID 인증용)
# TYPE 파일 유형 (unix → Unix Domain Socket)
## 8u → 8번 디스크립터, u = 읽기/쓰기
## 10u → 10번 디스크립터, u = 읽기/쓰기

# 유닉스 도메인 소켓 파일 정보 확인
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- ls -l /var/run/secrets/workload-spiffe-uds/socket
srw-rw-rw- 1 istio-proxy istio-proxy 0 May  1 23:23 /var/run/secrets/workload-spiffe-uds/socket

# istio 인증서 확인 : 
docker exec -it myk8s-control-plane istioctl proxy-config secret deploy/webapp.istioinaction
RESOURCE NAME     TYPE           STATUS     VALID CERT     SERIAL NUMBER                               NOT AFTER                NOT BEFORE
default           Cert Chain     ACTIVE     true           45287494908809645664587660443172732423      2025-05-03T16:13:14Z     2025-05-02T16:11:14Z
ROOTCA            CA             ACTIVE     true           338398148201570714444101720095268162852     2035-04-29T07:46:14Z     2025-05-01T07:46:14Z

docker exec -it myk8s-control-plane istioctl proxy-config secret deploy/webapp.istioinaction -o json
...

echo "." | base64 -d  | openssl x509 -in /dev/stdin -text -noout


# istio ca 관련
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- ls -l /var/run/secrets/istio
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- openssl x509 -in /var/run/secrets/istio/root-cert.pem -text -noout

kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- ls -l /var/run/secrets/tokens
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- cat /var/run/secrets/tokens/istio-token
TOKEN=$(kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- cat /var/run/secrets/tokens/istio-token)
# 헤더 디코딩 
echo $TOKEN | cut -d '.' -f1 | base64 --decode | sed 's/$/}/' | jq
# 페이로드 디코딩
echo $TOKEN | cut -d '.' -f2 | base64 --decode | sed 's/$/"}/' | jq
# (옵션) brew install jwt-cli 
jwt decode $TOKEN


# (참고) k8s ca 관련
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- ls -l /var/run/secrets
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- ls -l /var/run/secrets/kubernetes.io/serviceaccount
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- openssl x509 -in /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -text -noout
kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
TOKEN=$(kubectl exec -it -n istioinaction deploy/webapp -c istio-proxy -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)
# 헤더 디코딩 
echo $TOKEN | cut -d '.' -f1 | base64 --decode | sed 's/$/}/' | jq
# 페이로드 디코딩
echo $TOKEN | cut -d '.' -f2 | base64 --decode | sed 's/$/}/' | jq
# (옵션) brew install jwt-cli 
jwt decode $TOKEN


# (참고)
kubectl port-forward deploy/webapp -n istioinaction 15000:15000
open http://localhost:15000
curl http://localhost:15000/certs
```
- 이제 프록시는 클라이언트에게 자신의 정체를 증명할 수 있으며 상호 인증 커넥션을 시작할 수 있다.
- 그림 C.5는 이 **과정을 간략하게 요약**한 것이다.

![[Pasted image 20250817224944.png]]
1. **이스티오 프록시 컨테이너에 서비스 어카운트 토큰이 할당된다.**
2. **토큰과 CSR이 istiod로 전송된다.**
3. **istiod는 쿠버네티스 TokenReview API로 토큰의 유효성을 검사한다.**
4. **성공하면, 인증서에 서명하고 응답으로 제공한다.**
5. **파일럿 에이전트는 엔보이 SDS를 통해 엔보이가 ID를 포함한 인증서를 사용하도록 설정한다.**


- 그리고 이것이 이스티오가 워크로드ID를 프로비저닝하기 위해 SPIFFE 사양을 구현하는 전체 과정이다.
- 이 과정은 이스티오 프록시 사이드카가 주입되는 모든 워크로드에서 자동으로 수행된다.
![[Pasted image 20250817225010.png]]

## (참고)[사전 지식] K8S 파드의 애플리케이션이 사용할 수 있는 인증 관련 정보 - [Docs](https://kubernetes.io/ko/docs/reference/access-authn-authz/service-accounts-admin/) , [Link](https://ssup2.github.io/blog-old/theory_analysis/Kubernetes_Authentication_Service_Account/)
![[Pasted image 20250817225034.png]]
- **서비스 어카운트** **S**ervice **A**ccount
    - 서비스어카운트(ServiceAccount) 는 파드에서 실행되는 애플리케이션 프로세스에 대한 식별자를 제공한다.
    - 파드 내부의 애플리케이션 프로세스는, 자신에게 부여된 서비스 어카운트의 식별자를 사용하여 클러스터의 API 서버에 인증할 수 있다.
- **서비스 어카운트 토큰** **s**ervice**A**ccount**Token**
    - `서비스어카운트토큰(serviceAccountToken)` 정보는 kubelet이 kube-apiserver로부터 취득한 토큰을 포함한다.
    - kubelet은 TokenRequest API를 통해 일정 시간 동안 사용할 수 있는 토큰을 발급 받는다.
    - 이렇게 취득한 토큰은 파드가 삭제되거나 지정된 수명 주기 이후에 만료된다(기본값은 1시간이다).
    - 이 토큰은 특정한 파드에 바인딩되며 kube-apiserver를 그 대상으로 한다.
- **토큰 컨트롤러** token Controller
    - `kube-controller-manager` 의 일부로써 실행되며, 비동기적으로 동작한다.
    - 서비스어카운트에 대한 삭제를 감시하고, 해당하는 모든 서비스어카운트 토큰 시크릿을 같이 삭제한다.
    - 서비스어카운트 토큰 시크릿에 대한 추가를 감시하고, 참조된 서비스어카운트가 존재하는지 확인하며, 필요한 경우 시크릿에 토큰을 추가한다.
    - 시크릿에 대한 삭제를 감시하고, 필요한 경우 해당 서비스어카운트에서 참조 중인 항목들을 제거한다.
- **서비스 어카운트 어드미션 컨트롤러 S**ervice **A**ccount Admission Controller
    어드미션 컨트롤러는 파드의 생성 시점에 다음 작업들을 수행한다.
    1. 파드에 `.spce.serviceAccountName` 항목이 지정되지 않았다면, 어드미션 컨트롤러는 실행하려는 파드의 서비스어카운트 이름을 `default`로 설정한다.
    2. 어드미션 컨트롤러는 실행되는 파드가 참조하는 서비스어카운트가 존재하는지 확인한다.
        - 만약 해당하는 이름의 서비스어카운트가 존재하지 않는 경우, 어드미션 컨트롤러는 파드를 실행시키지 않는다.
        - 이는 `default` 서비스어카운트에 대해서도 동일하게 적용된다.
    3. 서비스어카운트의 `automountServiceAccountToken` 또는 파드의 `automountServiceAccountToken` 중 어느 것도 `false` 로 설정되어 있지 않다면,
        - 어드미션 컨트롤러는 실행하려는 파드에 API에 접근할 수 있는 토큰을 포함하는 [볼륨](https://kubernetes.io/ko/docs/concepts/storage/volumes/) 을 추가한다.
        - 어드미션 컨트롤러는 파드의 각 컨테이너에 `volumeMount`를 추가한다.
        - 이미 `/var/run/secrets/kubernetes.io/serviceaccount` 경로에 볼륨이 마운트 되어있는 컨테이너에 대해서는 추가하지 않는다.
        - 리눅스 컨테이너의 경우, 해당 볼륨은 `/var/run/secrets/kubernetes.io/serviceaccount` 위치에 마운트된다
    4. 파드의 spec에 `imagePullSecrets` 이 없는 경우, 어드미션 컨트롤러는 `ServiceAccount`의 `imagePullSecrets`을 복사하여 추가된다.
- **TokenRequest API**
    - 서비스어카운트의 하위 리소스인 [TokenRequest](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)를 사용하여 일정 시간 동안 해당 서비스어카운트에서 사용할 수 있는 토큰을 가져올 수 있다.
    - 컨테이너 내에서 사용하기 위한 API 토큰을 얻기 위해 이 요청을 직접 호출할 필요는 없는데, kubelet이 _프로젝티드 볼륨_ 을 사용하여 이를 설정하기 때문이다.
- **프로젝티드 볼륨** Projected Volumes - [Docs](https://kubernetes.io/ko/docs/concepts/storage/projected-volumes/)

# Cilium Mutual Authentication (Beta) 설정 : 실습 환경 구성 실패로 Skip - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/mutual-authentication/mutual-authentication/) & Mutual Authentication Example - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/mutual-authentication/mutual-authentication-example/)

**Mutual Authentication in Cilium**

- Cilium의 mTLS 기반 상호 인증 지원은 일반 연결에 대해 대역 외 상호 인증 핸드셰이크를 제공합니다.
    - Cilium’s mTLS-based Mutual Authentication support brings the mutual authentication handshake out-of-band for regular connections.
- Cilium이 서비스 간 인증 및 암호화에 대한 일반적인 요구 사항 대부분을 충족하려면 사용자가 암호화를 활성화해야 합니다.
    - For Cilium to meet most of the common requirements for service-to-service authentication and encryption, users must enable encryption.
    - Cilium의 암호화 기능인 [WireGuard Transparent Encryption](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/#encryption-wg) 및 [IPsec Transparent Encryption을](https://docs.cilium.io/en/stable/security/network/encryption-ipsec/#encryption-ipsec) 활성화하면 Pod 간에 암호화된 연결을 자동으로 생성하고 유지할 수 있습니다.
- 신원 확인의 과제를 해결하기 위해 상호 인증에는 분산 시스템을 위한 안전한 신원 확인 프레임워크가 필요합니다.
    - To address the challenge of identity verification in dynamic and heterogeneous environments, mutual authentication requires a framework secure identity verification for distributed systems.

**Identity Management**

- Cilium의 현재 상호 인증 지원에서는 SPIFFE(Secure Production Identity Framework for Everyone)를 사용하여 ID 관리가 제공됩니다.
    - In Cilium’s current mutual authentication support, identity management is provided through the use of **SPIFFE** (Secure Production Identity Framework for Everyone).
- [SPIFFE](https://spiffe.io/) 가 제공하는 혜택
    - **신뢰할 수 있는 신원 발급**: SPIFFE는 신원 발급 및 관리를 위한 표준화된 메커니즘을 제공합니다. 서비스가 빈번하게 확장되거나 축소되는 동적 환경에서도 분산 시스템의 각 서비스가 고유하고 검증 가능한 신원을 받을 수 있도록 보장합니다.
    - **신원 증명**: SPIFFE는 서비스가 증명을 통해 신원을 증명할 수 있도록 합니다. SPIFFE는 디지털 서명이나 암호화 증명과 같은 신원에 대한 검증 가능한 증거를 제공함으로써 서비스의 진위성과 무결성을 입증할 수 있도록 보장합니다.
    - **동적이고 확장 가능한 환경**: SPIFFE는 동적인 환경에서 ID 관리의 어려움을 해결합니다. 서비스가 지속적으로 배포, 업데이트 또는 폐기될 수 있는 클라우드 네이티브 아키텍처에서 필수적인 자동 ID 발급, 순환 및 폐기 기능을 지원합니다.
- Cilium and SPIFFE
    - SPIFFE는 워크로드가 중앙 서버에 ID를 요청할 수 있도록 하는 API 모델을 제공합니다. 이 경우 워크로드는 Cilium 보안 ID와 동일한 의미, 즉 레이블 집합으로 설명되는 포드 집합을 의미합니다. SPIFFE ID는 URI의 하위 클래스이며 다음과 같은 형태를 갖습니다.
        - `spiffe://trust.domain/path/with/encoded/info`
    - SPIFFE 설정에는 두 가지 주요 부분이 있습니다.
        - 신뢰 도메인에 대한 신뢰의 루트를 형성하는 **중앙 SPIRE 서버**입니다.
        - **노드별 SPIRE 에이전트**는 먼저 SPIRE 서버로부터 자체 ID를 받은 다음 해당 노드에서 실행 중인 워크로드의 ID 요청을 검증합니다.
    - 워크로드가 자신의 ID를 얻고자 할 때(일반적으로 시작 시) SPIFFE 워크로드 API를 사용하여 로컬 SPIRE 에이전트에 연결하고 에이전트에 자신을 설명합니다.
    - 그런 다음 SPIRE 에이전트는 워크로드가 실제로 표시된 것과 같은지 확인한 다음 SPIRE 서버에 연결하여 워크로드가 ID를 요청하고 있으며 요청이 유효함을 증명합니다.
    - SPIRE 에이전트는 작업 부하에 대해 여러 가지 사항을 확인합니다. 즉, 포드가 실제로 해당 노드에서 실행 중인지, 레이블이 일치하는지 등을 확인합니다.
    - SPIRE 에이전트가 SPIRE 서버에 신원을 요청하면, SVID(SPIFFE Verified Identity Document) 형식으로 워크로드에 다시 전달합니다. 이 문서에는 X.509 버전의 TLS 키 쌍이 포함되어 있습니다.
    - SPIRE의 일반적인 흐름에서 워크로드는 SPIRE 서버에 자체 정보를 요청합니다. Cilium의 SPIFFE 지원에서는 Cilium 에이전트가 공통 SPIFFE ID를 받고 다른 워크로드를 대신하여 ID를 요청할 수 있습니다.
- 필수 조건
    - 상호 인증은 현재 인증서 관리를 위한 SPIFFE API에서만 지원됩니다.
        - Mutual authentication is only currently supported with SPIFFE APIs for certificate management.
    - Cilium Helm 차트에는 상호 인증을 위해 SPIRE 서버를 배포하는 옵션이 포함되어 있습니다. 자체 SPIRE 서버를 배포하고 Cilium을 구성하여 사용할 수도 있습니다.
        - The Cilium Helm chart includes an option to deploy a SPIRE server for mutual authentication. You may also deploy your own SPIRE server and configure Cilium to use it.

**설정**

- 기본 설치에는 클러스터에서 [PersistentVolumeClaim 지원이 필요하므로 클러스터 공급자에게 해당 기능이 지원되는지 또는 활성화 방법을 확인하세요.](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- 랩 또는 로컬 클러스터의 경우 설치 명령에 전달하여 메모리 내 스토리지로 전환할 수 있지만 `authentication.mutual.spire.install.server.dataStorage.enabled=false` , SPIRE 서버 포드를 다시 시작할 때 모든 데이터를 다시 생성해야 합니다.

제한 사항

- Cilium 상호 인증은 아직 개발 중이며 베타 버전으로 제공됩니다. 계획된 여러 보안 기능이 아직 구현되지 않았습니다. 자세한 내용은 아래를 참조하세요.
- Cilium의 상호 인증은 SPIFFE의 프로덕션 지원 구현체인 SPIRE로만 검증되었습니다. Cilium은 SPIFFE API를 사용하므로 다른 SPIFFE 구현체도 작동할 수 있습니다. 그러나 Cilium은 현재 제공된 SPIRE 설치 환경에서만 테스트되었으며, 다른 SPIFFE 구현체는 현재 지원되지 않습니다.
- 현재 클러스터 메시와 서비스 메시를 결합하기 위해 여러 클러스터에 걸쳐 단일 신뢰 도메인을 구축하는 옵션은 없습니다. 따라서 클러스터 메시에 연결된 클러스터는 현재 상호 인증과 호환되지 않습니다.
- 현재 상호 인증 지원은 Cilium에서 관리하는 클러스터 내에서만 작동하며 외부 mTLS 솔루션과 호환되지 않습니다.

```bash
#
helm get values -n kube-system cilium > before.yaml
helm upgrade cilium cilium/cilium --version 1.18.1 --namespace kube-system --reuse-values \
--set authentication.mutual.spire.enabled=true --set authentication.mutual.spire.install.enabled=true

helm get values -n kube-system cilium > after.yaml
diff before.yaml after.yaml

#
kc describe cm -n cilium-spire spire-server | grep socket_path
  socket_path = "/tmp/spire-server/private/api.sock"

# k9s 에서 configmap 수정
socket_path = "/run/spire/sockets/api.sock"  

# 
kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart ds/cilium
kubectl -n cilium-spire rollout restart sts spire-server

# 확인
kubectl get all,svc,ep,configmap,secret,pvc -n cilium-spire

# This enabled the mutual authentication feature and automatically deployed a SPIRE server.
## If you're not familiar with SPIFFE and SPIRE, don't worry: we will explain the concepts behind SPIFFE and SPIRE (and how they integrate with Cilium) - in a later task.
## Just know that SPIFFE is an identity framework for identifying and securing communications between microservices while SPIRE is a production-ready implementation of SPIFFE.
cilium config view | grep mesh-auth
mesh-auth-enabled                                 true
mesh-auth-gc-interval                             5m0s
mesh-auth-mutual-connect-timeout                  5s
mesh-auth-mutual-enabled                          true
mesh-auth-mutual-listener-port                    4250
mesh-auth-queue-size                              1024
mesh-auth-rotated-identities-queue-size           1024
mesh-auth-spiffe-trust-domain                     spiffe.cilium
mesh-auth-spire-admin-socket                      /run/spire/sockets/admin.sock
mesh-auth-spire-agent-socket                      /run/spire/sockets/agent/agent.sock
mesh-auth-spire-server-address                    spire-server.cilium-spire.svc:8081
mesh-auth-spire-server-connection-timeout         30s

# You can see above that:
## Mutual Authentication is enabled.
## the port where mutual handshakes between agents will be performed is by default set to 4250.
## the SPIFFE Trust Domain is set to spiffe.cilium by default.
## the SPIRE connection timeout is set to 30s by default.
## Note that the default SPIRE settings can be customized at deployment.

# Note that, while Mutual Authentication has been enabled globally, it won't apply to workloads until a network policy applicable to these workloads has the authentication.mode:required setting. You will learn more about this in the upcomign tasks.

# Let's also debug log level, as this will be useful to show the actual mutual authentication later on:
cilium config set debug true
```
# CFP-22215: Mutual Authentication for Cilium Service Mesh* - [Github](https://github.com/cilium/design-cfps/blob/main/cilium/CFP-22215-mutual-auth-for-service-mesh.md) , [Blog](https://isovalent.com/blog/post/2022-05-03-servicemesh-security/)
![[Pasted image 20250817225217.png]]
![[Pasted image 20250817225226.png]]
![[Pasted image 20250817225235.png]]
- 목표
    - 포드가 피어 인증을 선택할 수 있는 메커니즘을 제공합니다.
    - 기존 인증서 관리 시스템(SPIFFE, Vault, SMI, Istio, cert-manager 등)에 플러그인 가능
    - 구성 가능한 인증서 세분성
    - 데이터플레인에 대한 기존 암호화 프로토콜 지원 활용
    - 합리적인 노력으로 가능한 한 패킷 삭제를 최소화합니다.
- 개요
    - 첫째, 각 노드의 cilium-agent 인스턴스 간의 제어 평면 연결은 노드 내 포드 간 연결 인증을 제공합니다.
    - 둘째, WireGuard 또는 IPsec을 사용하는 기존 Cilium 암호화 지원은 연결에 암호화된 데이터 평면을 제공합니다.
    - 본 문서에서는 이러한 대역 외 인증 채널과 대역 내 네트워크 암호화 채널을 조정하는 노드별 구현 방식을 설명.
- 아키텍처
![[Pasted image 20250817225306.png]]
![[Pasted image 20250817225320.png]]
SPIFEE Architecture
![[Pasted image 20250817225341.png]]

# Mutual Authentication with Cilium Online Lab - [Link](https://isovalent.com/labs/cilium-mutual-authentication/)
![[Pasted image 20250817225410.png]]
# Cilium Mutual Authentication 설정 확인

```bash
#
kubectl get node -owide
NAME                 STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION    CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane   52m   v1.31.0   172.18.0.2    <none>        Debian GNU/Linux 12 (bookworm)   6.11.0-1015-gcp   containerd://1.7.18
kind-worker          Ready    <none>          52m   v1.31.0   172.18.0.3    <none>        Debian GNU/Linux 12 (bookworm)   6.11.0-1015-gcp   containerd://1.7.18
kind-worker2         Ready    <none>          52m   v1.31.0   172.18.0.4    <none>        Debian GNU/Linux 12 (bookworm)   6.11.0-1015-gcp   containerd://1.7.18

# Cilium was installed during the lab boot-up and was deployed with the following Helm flags:
authentication:
  mutual:
    spire:
      enabled: true
      install:
        enabled: true

helm list -n kube-system
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
cilium  kube-system     1               2025-08-17 07:37:37.468306607 +0000 UTC deployed        cilium-1.17.4   1.17.4  

helm get values -n kube-system cilium
authentication:
  mutual:
    spire:
      enabled: true
      install:
        enabled: true
cluster:
  name: kind-kind
hubble:
  enabled: true
  metrics:
    enabled:
    - dns
    - drop
    - tcp
    - flow
    - icmp
    - http
  relay:
    enabled: true
    service:
      type: NodePort
  ui:
    enabled: true
    service:
      type: NodePort
ipam:
  mode: kubernetes
k8sServiceHost: 172.18.0.2
k8sServicePort: 6443
kubeProxyReplacement: true
operator:
  replicas: 1
routingMode: tunnel
securityContext:
  privileged: true
tunnelProtocol: vxlan

# 기능 활성화, SPIRE(SPIFFE의 구현체) 서버 배포됨
# his enabled the mutual authentication feature and automatically deployed a SPIRE server.
cilium config view | grep mesh-auth
mesh-auth-enabled                                 true
mesh-auth-gc-interval                             5m0s
mesh-auth-mutual-connect-timeout                  5s
mesh-auth-mutual-enabled                          true
mesh-auth-mutual-listener-port                    4250  # the port where mutual handshakes between agents will be performed is by default set to 4250.
mesh-auth-queue-size                              1024
mesh-auth-rotated-identities-queue-size           1024
mesh-auth-spiffe-trust-domain                     spiffe.cilium  # the SPIFFE Trust Domain is set to spiffe.cilium by default.
mesh-auth-spire-admin-socket                      /run/spire/sockets/admin.sock
mesh-auth-spire-agent-socket                      /run/spire/sockets/agent/agent.sock
mesh-auth-spire-server-address                    spire-server.cilium-spire.svc:8081
mesh-auth-spire-server-connection-timeout         30s  # the SPIRE connection timeout is set to 30s by default.

#
kubectl get pod,svc,ep -n cilium-spire
NAME                    READY   STATUS    RESTARTS   AGE
pod/spire-agent-pcrcr   1/1     Running   0          7m42s
pod/spire-agent-wvnr8   1/1     Running   0          7m42s
pod/spire-agent-xx8pf   1/1     Running   0          7m42s
pod/spire-server-0      2/2     Running   0          7m42s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/spire-server   ClusterIP   10.96.12.133   <none>        8081/TCP   7m42s

NAME                     ENDPOINTS           AGE
endpoints/spire-server   10.244.2.109:8081   7m42s


# Let's also debug log level, as this will be useful to show the actual mutual authentication later on:
cilium config set debug true
✨ Patching ConfigMap cilium-config with debug=true...
♻️  Restarted Cilium pods
```

# Deploy the demo
```bash
# 스타워즈 환경과 데모에 사용된 L3/L4 네트워크 정책을 배포, 아직 상호인증 X
# In this example, we will look at adding mutual authentication to the Star Wars demo deployed in the Getting Started with the Star Wars Demo docs and available in the Getting Started with Cilium lab.
# We will assume you are familiar with this demo. If not, check the links above.
# Deploy the Star Wars environment and the L3/L4 network policy used in the demo (there's no mutual authentication in the network policy yet).
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/http-sw-app.yaml
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/sw_l3_l4_policy.yaml

#
kubectl get pod,svc,ep
NAME                            READY   STATUS    RESTARTS   AGE
pod/deathstar-67c5c5c88-m7nrc   1/1     Running   0          5m43s
pod/deathstar-67c5c5c88-ng7d8   1/1     Running   0          5m43s
pod/tiefighter                  1/1     Running   0          5m43s
pod/xwing                       1/1     Running   0          5m43s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/deathstar    ClusterIP   10.96.189.159   <none>        80/TCP    5m43s
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   51m

NAME                   ENDPOINTS                         AGE
endpoints/deathstar    10.244.2.204:80,10.244.2.226:80   5m43s
endpoints/kubernetes   172.18.0.2:6443                   51m


# Review the Network Policy
## This network policy will only allow traffic from endpoints labeled with org=empire to endpoints with both the class=deathstar and org=empire labels, over TCP port 80.
kubectl get cnp rule1 -o yaml | yq .spec
description: L3-L4 policy to restrict deathstar access to empire ships only
endpointSelector:
  matchLabels:
    class: deathstar
    org: empire
ingress:
  - fromEndpoints:
      - matchLabels:
          org: empire
    toPorts:
      - ports:
          - port: "80"
            protocol: TCP
```

# Check Network Policy
![[Pasted image 20250817225449.png]]
```bash
# verify that Tie Fighters (Empire space ships) are allowed to land on the Death Star : 성공!
# This request should be successful (the tiefighter is able to connect to the deathstar over a specific HTTP path):
kubectl exec tiefighter -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

# Then verify that the X-Wing ship (belong to the Alliance) is denied access to the Death Star: 실패!
# This second request should time out (the xwing is unable to connect to the deathstar because it does not have the right label):
kubectl exec xwing -- \
  curl -s --connect-timeout 1 -XPOST deathstar.default.svc.cluster.local/v1/request-landing
command terminated with exit code 28

#
cilium hubble port-forward&
hubble observe --pod tiefighter
hubble observe --pod xwing
Aug 17 07:52:56.011: default/xwing:50210 (ID:30862) <> default/deathstar-67c5c5c88-ng7d8:80 (ID:3838) policy-verdict:none INGRESS DENIED (TCP Flags: SYN)
Aug 17 07:52:56.011: default/xwing:50210 (ID:30862) <> default/deathstar-67c5c5c88-ng7d8:80 (ID:3838) Policy denied DROPPED (TCP Flags: SYN)
```

# Enforcing Mutual Authentication*
```bash
# CNP에 상호 인증 적용
spec:
  egress|ingress:
    authentication:
        mode: "required"

# Let's do that now. We will be using this policy:
yq sw_l3_l4_l7_mutual_authentication_policy.yaml
---
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "Mutual authentication enabled L7 policy"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
    - fromEndpoints:
        - matchLabels:
            org: empire
      authentication:
        mode: "required"
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:
              - method: "POST"
                path: "/v1/request-landing"

# Review the changes we will be making with the existing network policy
KUBECTL_EXTERNAL_DIFF='colordiff -u' \
  kubectl diff -f sw_l3_l4_l7_mutual_authentication_policy.yaml | \
  grep -A30 ' spec:'
 spec:
-  description: L3-L4 policy to restrict deathstar access to empire ships only
+  description: Mutual authentication enabled L7 policy
   endpointSelector:
     matchLabels:
       class: deathstar
       org: empire
   ingress:
-  - fromEndpoints:
+  - authentication:
+      mode: required  # ingress access is only for mutually authenticated workloads.
+    fromEndpoints:
     - matchLabels:
         org: empire
     toPorts:
     - ports:
       - port: "80"
         protocol: TCP
+      rules:
+        http:
+        - method: POST
+          path: /v1/request-landing  # we are adding L7 filtering (only allowing HTTP POST to the /v1/request-landing)
 status:
   conditions:
   - lastTransitionTime: "2025-08-17T07:39:18Z"

# Let's now apply this policy : 기존 정책에 업데이트 적용.
kubectl apply -f sw_l3_l4_l7_mutual_authentication_policy.yaml
ciliumnetworkpolicy.cilium.io/rule1 configured
```

## 상호 인증 설정 후 호출 확인
```bash
# Re-try the connectivity tests : Let's start with the tiefighter calling the /request-landing path :
# This should still succeed. It may take some time for the response to print in the terminal.
kubectl exec tiefighter -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

# Let's then try access from the tiefigher to the /exhaust-port path:
# This second request should be denied , thanks to the new L7 Network Policy, preventing any tiefighter - compromised or not - from accessing the /exhaust-port.
kubectl exec tiefighter -- \
  curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Access denied

# This third one should time out, thanks to the L3/L4 Network Policy.
# But has mutual authentication actually happened?
# Let's verify this in the next challenge.
kubectl exec xwing -- \
  curl -s --connect-timeout 1 -XPOST deathstar.default.svc.cluster.local/v1/request-landing
command terminated with exit code 28

#
hubble observe --pod tiefighter
Aug 17 08:00:51.951: default/tiefighter:46720 (ID:3712) -> default/deathstar-67c5c5c88-m7nrc:80 (ID:3838) http-request DROPPED (HTTP/1.1 PUT http://deathstar.default.svc.cluster.local/v1/exhaust-port)

hubble observe --pod xwing
Aug 17 08:01:29.601: default/xwing:44562 (ID:30862) <> default/deathstar-67c5c5c88-m7nrc:80 (ID:3838) policy-verdict:none INGRESS DENIED (TCP Flags: SYN)
Aug 17 08:01:29.601: default/xwing:44562 (ID:30862) <> default/deathstar-67c5c5c88-m7nrc:80 (ID:3838) Policy denied DROPPED (TCP Flags: SYN)
```

## Hubble 확인
```bash
#
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

kubectl exec xwing -- curl -s --connect-timeout 1 -XPOST deathstar.default.svc.cluster.local/v1/request-landing
command terminated with exit code 28

# xwing 은 L3/L4 정책으로 차단!
# The network policy should have dropped flows from the xwing as the xwing has not got the right labels.
# The policy verdict for this traffic should be DROPPED by the L3/L4 section of the Network Policy:
hubble observe --type drop --from-pod default/xwing
Aug 17 08:01:29.601: default/xwing:44562 (ID:30862) <> default/deathstar-67c5c5c88-m7nrc:80 (ID:3838) Policy denied DROPPED (TCP Flags: SYN)
Aug 17 08:03:48.946: default/xwing:58126 (ID:30862) <> default/deathstar-67c5c5c88-ng7d8:80 (ID:3838) Policy denied DROPPED (TCP Flags: SYN)


# tiefighter 는 첫 번째 패킷 차단 유도되고, 이후 인증 단계를 진행하려함!
# Let's now look at traffic from the tiefighter to the deathstar.
# The network policy should have dropped the first flow from the tiefighter to the deathstar Service over /request-landing. Why ? Because the first packet to match the mutual authentication-based network policy will kickstart the mutual authentication handshake.
hubble observe --type drop --from-pod default/tiefighter
Jul  4 08:30:19.341: default/tiefighter:43412 (ID:10685) <> default/deathstar-f694cf746-5wxks:80 (ID:53245) Authentication required DROPPED (TCP Flags: SYN)

hubble observe --type policy-verdict --from-pod default/tiefighter
Aug 17 08:05:12.609: default/tiefighter:36570 (ID:3712) -> default/deathstar-67c5c5c88-m7nrc:80 (ID:3838) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN; Auth: SPIRE)
Aug 17 08:05:14.747: default/tiefighter:36304 (ID:3712) -> default/deathstar-67c5c5c88-m7nrc:80 (ID:3838) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN; Auth: SPIRE)
Aug 17 08:05:18.043: default/tiefighter:36310 (ID:3712) -> default/deathstar-67c5c5c88-m7nrc:80 (ID:3838) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN; Auth: SPIRE)
```

# SPIRE & SPIFFE*
- A **SPIRE** server was automatically deployed when installing Cilium with the mutual authentication feature.
- The **SPIRE** environment will **manage** the **TLS certificates** for the workloads managed by Cilium.
    
## Verify SPIRE Health
```bash
# spire server와 agent 확인
kubectl get all -n cilium-spire
NAME                    READY   STATUS    RESTARTS   AGE
pod/spire-agent-pcrcr   1/1     Running   0          34m
pod/spire-agent-wvnr8   1/1     Running   0          34m
pod/spire-agent-xx8pf   1/1     Running   0          34m
pod/spire-server-0      2/2     Running   0          34m

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/spire-server   ClusterIP   10.96.12.133   <none>        8081/TCP   34m

NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/spire-agent   3         3         3       3            3           <none>          34m

NAME                            READY   AGE
statefulset.apps/spire-server   1/1     34m

# Let's run a healthcheck on the SPIRE server.
# Expect a healthy response:
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- /opt/spire/bin/spire-server -h

kubectl exec -n cilium-spire spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server healthcheck
Server is healthy.

# Let's verify the list of SPIRE agents: 3개의 노드에 배포된 SPIRE agent 확인
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server agent list
Found 3 attested agents:

SPIFFE ID         : spiffe://spiffe.cilium/spire/agent/k8s_psat/kind-kind/a3339ad9-41ac-43f2-9c59-ee1456812151
Attestation type  : k8s_psat
Expiration time   : 2025-08-17 09:05:42 +0000 UTC
Serial number     : 182354946325480289712988905712661761955
Can re-attest     : true

SPIFFE ID         : spiffe://spiffe.cilium/spire/agent/k8s_psat/kind-kind/0a6c3ffd-2542-427a-b236-bb188157d836
Attestation type  : k8s_psat
Expiration time   : 2025-08-17 09:06:08 +0000 UTC
Serial number     : 331299735369757762765377339454747741309
Can re-attest     : true

SPIFFE ID         : spiffe://spiffe.cilium/spire/agent/k8s_psat/kind-kind/68d3278e-d15a-4c9c-a15c-3a5e3fff4a12
Attestation type  : k8s_psat
Expiration time   : 2025-08-17 09:06:36 +0000 UTC
Serial number     : 186656089107650682960504533299183399838
Can re-attest     : true
```
SPIRE 서버는 Kubernetes 클러스터에서 실행되는 SPIRE 에이전트의 신원을 확인하기 위해 Kubernetes Projected Service Account Token(PSAT)을 사용합니다.

## Verify SPIFFE identity for Cilium
```bash
# cilium-agent 와 operator 이 SPIRE 서버에 등록된 위임 ID 보유
# First, verify that the Cilium agent and operator have identities on the SPIRE server:
# the Cilium agent and the Cilium Operator, showing that the Cilium agent and operator each have a registered delegate identity with the SPIRE Server
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server entry show -parentID spiffe://spiffe.cilium/ns/cilium-spire/sa/spire-agent
Found 2 entries
Entry ID         : eb3af2eb-0936-4ea5-ab26-ec9abdc98507
SPIFFE ID        : spiffe://spiffe.cilium/cilium-agent
Parent ID        : spiffe://spiffe.cilium/ns/cilium-spire/sa/spire-agent
Revision         : 0
X509-SVID TTL    : default
JWT-SVID TTL     : default
Selector         : k8s:ns:kube-system
Selector         : k8s:sa:cilium

Entry ID         : 3e48a856-6163-42f7-9307-6a165b748baa
SPIFFE ID        : spiffe://spiffe.cilium/cilium-operator
Parent ID        : spiffe://spiffe.cilium/ns/cilium-spire/sa/spire-agent
Revision         : 0
X509-SVID TTL    : default
JWT-SVID TTL     : default
Selector         : k8s:ns:kube-system
Selector         : k8s:sa:cilium-operator

# Let's now verify that the Cilium operator has registered identities with the SPIRE server on behalf of the workloads (Kubernetes Pods).
```

## Verify SPIFFE identity for the Death Star
```bash
# First, get the Cilium Identity of the deathstar Pods:
# Even though there are two of these Pods, they share the same Cilium Identity, since they use the same set of Kubernetes labels.
kubectl get cep -owide
NAME                        SECURITY IDENTITY   INGRESS ENFORCEMENT   EGRESS ENFORCEMENT   ENDPOINT STATE   IPV4           IPV6
deathstar-67c5c5c88-m7nrc   3838                                                           ready            10.244.2.204   
deathstar-67c5c5c88-ng7d8   3838                                                           ready            10.244.2.226   
tiefighter                  3712                                                           ready            10.244.2.17    
xwing                       30862                                                          ready            10.244.2.65  
...

IDENTITY_ID=$(kubectl get cep -l app.kubernetes.io/name=deathstar -o=jsonpath='{.items[0].status.identity.id}')
echo $IDENTITY_ID
3838

# The SPIFFE ID —that uniquely identifies a workload— is based on the Cilium identity.
# It follows the spiffe://spiffe.cilium/identity/$IDENTITY_ID format.

# cilium-operator 가 Cilium identiy 를 기반으로 SPIRE Entry 생성 관리 책임 가짐!
# Verify that the Death Star pods have a registered SPIFFE identity on the SPIRE server:
# You can see the that the cilium-operator was listed as the Parent ID. 
# That is because the Cilium operator is responsible for creating SPIRE entries for each Cilium identity.
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server entry show -spiffeID spiffe://spiffe.cilium/identity/$IDENTITY_ID
Found 1 entry
Entry ID         : e8eddb0a-6152-4a17-8e08-fee936467ac1
SPIFFE ID        : spiffe://spiffe.cilium/identity/3838
Parent ID        : spiffe://spiffe.cilium/cilium-operator
Revision         : 0
X509-SVID TTL    : default
JWT-SVID TTL     : default
Selector         : cilium:mutual-auth
 
# List all the registration entries with:
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server entry show -selector cilium:mutual-auth
Found 9 entries
Entry ID         : 19dd98c3-4934-4ac1-9ff9-37b761ae7c83
SPIFFE ID        : spiffe://spiffe.cilium/identity/10101
Parent ID        : spiffe://spiffe.cilium/cilium-operator
Revision         : 0
X509-SVID TTL    : default
JWT-SVID TTL     : default
Selector         : cilium:mutual-auth

Entry ID         : ea5a90bd-27b1-488f-914c-de1fe2eee167
SPIFFE ID        : spiffe://spiffe.cilium/identity/16058
Parent ID        : spiffe://spiffe.cilium/cilium-operator
Revision         : 0
X509-SVID TTL    : default
JWT-SVID TTL     : default
Selector         : cilium:mutual-auth

Entry ID         : 48cda74c-0e21-4dee-ac60-297fed739a7f
SPIFFE ID        : spiffe://spiffe.cilium/identity/19452
Parent ID        : spiffe://spiffe.cilium/cilium-operator
Revision         : 0
X509-SVID TTL    : default
JWT-SVID TTL     : default
Selector         : cilium:mutual-auth
...

# There are as many entries as there are identities. Verify this these match by listing the Cilium identities in the cluster:
kubectl get ciliumidentities
NAME    NAMESPACE            AGE
10101   cilium-spire         44m
16058   kube-system          44m
19452   kube-system          44m
30862   default              43m
34226   kube-system          44m
3712    default              43m
3838    default              43m
41907   local-path-storage   44m
46205   local-path-storage   44m
```

- **SPIFFE Verifiable Identity Document (SVID)**
    - SVID는 워크로드가 리소스나 호출자에게 자신의 신원을 증명하는 문서입니다. SVID는 SPIFF ID의 신뢰 도메인 내에서 권한자가 서명한 경우 유효한 것으로 간주됩니다.
        - _An `SVID` is the document with which a workload proves its identity to a resource or caller. An SVID is considered valid if it has been signed by an authority within the SPIFFE ID’s trust domain._
    - SVID에는 단일 SPIFF ID가 포함되어 있으며, 이는 해당 서비스를 제공하는 서비스의 신원을 나타냅니다. SVID는 X.509 인증서의 암호학적으로 검증 가능한 문서에 SPIFF ID를 인코딩합니다.
        - _An SVID contains a single SPIFFE ID, which represents the identity of the service presenting it. It encodes the SPIFFE ID in a cryptographically-verifiable document in an X.509 certificate._
    - Cilium 상호 인증의 초기 구현을 위해 SPIRE를 선택한 이유 중 하나는 인증서가 만료되기 전에 SVID를 자동으로 재키핑하고, 이 경우 실륨 에이전트를 포함한 SVID 감시자에게 통지하기 때문입니다.
        - _One of the reasons for choosing SPIRE for the initial implementation of Cilium Mutual Authentication is that it will automatically rekey SVIDs before their certificate expires, and when this happens, it will notify SVID watchers, which includes the Cilium Agent._

# Review Mutual Authentication Process

```bash
# workers 노드의 cilium-agent 이름 확인
CILIUM_KIND_WORKER=$(kubectl -n kube-system get pods -l k8s-app=cilium -o jsonpath='{.items[?(@.spec.nodeName=="kind-worker")].metadata.name}')
echo $CILIUM_KIND_WORKER
CILIUM_KIND_WORKER2=$(kubectl -n kube-system get pods -l k8s-app=cilium -o jsonpath='{.items[?(@.spec.nodeName=="kind-worker2")].metadata.name}')
echo $CILIUM_KIND_WORKER2
cilium-2mslq
cilium-vw752

# cilium-agent 로그에서 상호 인증 관련 내용 확인 가능
kubectl -n kube-system -c cilium-agent logs $CILIUM_KIND_WORKER --timestamps=true | grep "Policy is requiring authentication\|Validating Server SNI\|Validated certificate\|Successfully authenticated"
kubectl -n kube-system -c cilium-agent logs $CILIUM_KIND_WORKER2 --timestamps=true | grep "Policy is requiring authentication\|Validating Server SNI\|Validated certificate\|Successfully authenticated"
2025-08-17T08:00:24.806708742Z time=2025-08-17T08:00:24Z level=debug msg="Policy is requiring authentication" module=agent.controlplane.auth key="localIdentity=3838, remoteIdentity=3712, remoteNodeID=0, authType=spire"
2025-08-17T08:00:24.863381354Z time=2025-08-17T08:00:24Z level=debug msg="Validating Server SNI" module=agent.controlplane.auth SNI_ID=3712
2025-08-17T08:00:24.863406574Z time=2025-08-17T08:00:24Z level=debug msg="Validated certificate" module=agent.controlplane.auth uri-san=[spiffe://spiffe.cilium/identity/3712]
2025-08-17T08:00:24.863805612Z time=2025-08-17T08:00:24Z level=debug msg="Successfully authenticated" module=agent.controlplane.auth key="localIdentity=3838, remoteIdentity=3712, remoteNodeID=0, authType=spire" remote_node_ip=172.18.0.3
2025-08-17T08:07:14.365475019Z time=2025-08-17T08:07:14Z level=debug msg="Policy is requiring authentication" module=agent.controlplane.auth key="localIdentity=3838, remoteIdentity=3712, remoteNodeID=0, authType=spire"
2025-08-17T08:07:14.368235810Z time=2025-08-17T08:07:14Z level=debug msg="Validating Server SNI" module=agent.controlplane.auth SNI_ID=3712
2025-08-17T08:07:14.368256347Z time=2025-08-17T08:07:14Z level=debug msg="Validated certificate" module=agent.controlplane.auth uri-san=[spiffe://spiffe.cilium/identity/3712]
2025-08-17T08:07:14.368687549Z time=2025-08-17T08:07:14Z level=debug msg="Successfully authenticated" module=agent.controlplane.auth key="localIdentity=3838, remoteIdentity=3712, remoteNodeID=0, authType=spire" remote_node_ip=172.18.0.3
2025-08-17T08:07:19.174238640Z time=2025-08-17T08:07:19Z level=debug msg="Policy is requiring authentication" module=agent.controlplane.auth key="localIdentity=3838, remoteIdentity=3712, remoteNodeID=0, authType=spire"
2025-08-17T08:07:19.176168425Z time=2025-08-17T08:07:19Z level=debug msg="Validating Server SNI" module=agent.controlplane.auth SNI_ID=3712
2025-08-17T08:07:19.176176693Z time=2025-08-17T08:07:19Z level=debug msg="Validated certificate" module=agent.controlplane.auth uri-san=[spiffe://spiffe.cilium/identity/3712]
2025-08-17T08:07:19.176489738Z time=2025-08-17T08:07:19Z level=debug msg="Successfully authenticated" module=agent.controlplane.auth key="localIdentity=3838, remoteIdentity=3712, remoteNodeID=0, authType=spire" remote_node_ip=172.18.0.3
```

1. **네트워크 정책 생성**
    - A Network Policy with `authentication.mode: required` was created and will apply to traffic between identity `tiefighter` and identity `deathstar`.
2. **첫 번째 패킷은 차단되고, 상호 인증 프로세스 시작 알림 전송**
    - First packet from `tiefighter` to `deathstar` is dropped and Cilium is notified to start the mutual authentication process. Further packets will be dropped until mutual auth has completed. (`Policy is requiring authentication` log)
3. **Cilium 에이전트는 'tiefighter'의 ID를 가져오고, 'deathstar' 포드가 실행 중인 노드에 연결하여 상호 TLS 인증 핸드셰이크를 수행**
    - The Cilium agent retrieves the identity for `tiefighter`, connects to the node where the `deathstar` pod is running and performs a mutual TLS authentication handshake. (`Validating Server SNI` and `Validated certificate` logs)
4. **상호 인증 성공 완료 후 네트워크 정책이 제거되거나 항목이 만료될 때까지 '타이트파이터'에서 '데스스타'로 패킷 전달 통과**
    - When the handshake is successful (`Successfully authenticated` log), mutual authentication is now complete, and packets from `tiefighter` to `deathstar` will now flow until the network policy is removed or the entry expires (which is when the certificate does).

# 도전과제3 - Cilium Mutual Authentication Example 실습을 정상 진행 후 정리 - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/mutual-authentication/mutual-authentication-example/)

^53803d

