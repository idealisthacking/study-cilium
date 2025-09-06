# Cilium 제공 보안 : Identity-Based(Layer3), Port Level(Layer4), Application protocol Level(Layer7) - [Docs](https://docs.cilium.io/en/stable/security/network/intro/)

![[Pasted image 20250906222545.png]]

- 보안 정책은 세션 기반 프로토콜에 대해 **상태 저장 정책** 적용되어, **응답** 패킷은 자동으로 **허용**됨 - [Docs](https://docs.cilium.io/en/stable/security/network/policyenforcement/#policy-enforcement)
- 보안 정책은 **수신 or 송신** 시 적용됨
- **기본 보안 정책!** - [Docs](https://docs.cilium.io/en/stable/security/network/policyenforcement/#default-security-policy)
    - 정책이 로드되지 않은 경우, 정책 적용이 명시적으로 활성화되지 않은 한 모든 통신을 허용하는 것이 기본 동작입니다.
    - 첫 번째 정책 규칙이 로드되는 즉시 정책 적용이 자동으로 활성화되며, 모든 통신은 허용 목록에 추가되어야 하며, 그렇지 않으면 관련 패킷이 삭제됩니다.
    - 마찬가지로, 엔드포인트에 _L4_ 정책이 적용되지 않으면 모든 포트와의 통신이 허용됩니다.
    - 엔드포인트에 하나 이상의 _L4 정책을 연결하면 명시적으로 허용하지 않는 한 포트에 대한 모든 연결이 차단됩니다._

* Envoy Proxy - [Docs](https://docs.cilium.io/en/stable/security/network/proxy/envoy/)
![[Pasted image 20250906222619.png]]

- **투명한 암호화** : IPSec or WireGuard 로 Cilium 관리하는 호스트 트래픽과 엔드포인트 간 트래픽의 투명한 암호화 지원 - [Docs](https://docs.cilium.io/en/stable/security/network/encryption/) , [Youtube](https://www.youtube.com/watch?v=vj7M-t9MK6s)
    - IPSec : Limit(BPF 호스트 라우팅에서는 미동작, IPsec 터널당 단일 CPU 코어로 제한) - [Docs](https://docs.cilium.io/en/stable/security/network/encryption-ipsec/)
    - WireGuard : 터널 모드는 두 번 캡슐화됨 - [Docs](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/) , [Youtube](https://www.youtube.com/watch?v=-awkPi3D60E)
![[Pasted image 20250906222634.png]]

# Identity : 모든 엔드포인트에 ID가 할당. ID는 Labels 과 클러스터 내에 유일한 ID로 구성. - [Docs](https://docs.cilium.io/en/stable/gettingstarted/terminology/#identity)

- 엔드포인트에는 Security Relevant Labels에 일치하는 ID가 할당.
- 엔드포인트들이 동일한 Security Relevant Labels 사용 시 동일한 ID를 공유.

```bash
kubectl get ciliumendpoints.cilium.io -n kube-system
NAME                              SECURITY IDENTITY   ENDPOINT STATE   IPV4           IPV6
coredns-674b8bbfcf-7v8gz          14735               ready            172.20.0.138   
coredns-674b8bbfcf-sbftb          14735               ready            172.20.0.128   
hubble-relay-fdd49b976-fps92      15715               ready            172.20.0.32    
hubble-ui-655f947f96-cmncv        12634               ready            172.20.0.8     
metrics-server-5dd7b49d79-nx56l   8836                ready            172.20.0.49 

kubectl get ciliumidentities.cilium.io 
NAME    NAMESPACE            AGE
12634   kube-system          9m31s
14215   cilium-monitoring    9m31s
14735   kube-system          9m31s
15715   kube-system          9m31s
42930   local-path-storage   9m31s
58553   cilium-monitoring    9m31s
8836    kube-system          9m31s
```

* What is an Identity?
	- 엔드포인트의 ID는 [엔드포인트](https://docs.cilium.io/en/stable/gettingstarted/terminology/#endpoint) 에서 파생된 포드 또는 컨테이너와 연관된 [레이블을](https://docs.cilium.io/en/stable/gettingstarted/terminology/#labels) 기반으로 결정됩니다.
	- **파드 또는 컨테이너가 시작**되면 Cilium은 **컨테이너 런타임**에서 수신한 **이벤트**를 기반으로 네트워크에서 포드 또는 컨테이너를 나타내는 [엔드포인트](https://docs.cilium.io/en/stable/gettingstarted/terminology/#endpoint) 를 생성합니다 .
	- 다음 단계로 Cilium은 생성된 [엔드포인트](https://docs.cilium.io/en/stable/gettingstarted/terminology/#endpoint) 의 ID를 확인합니다 . 포드 또는 컨테이너의 [레이블이](https://docs.cilium.io/en/stable/gettingstarted/terminology/#labels) 변경될 때마다 ID가 재확인되고 필요에 따라 자동으로 수정됩니다.
```bash
kubectl get ciliumidentities.cilium.io 14735 -o yaml | yq
{
  "apiVersion": "cilium.io/v2",
  "kind": "CiliumIdentity",
  "metadata": {
    "creationTimestamp": "2025-08-31T00:47:09Z",
    "generation": 1,
    "labels": {
      "io.kubernetes.pod.namespace": "kube-system"
    },
    "name": "14735",
    "resourceVersion": "808",
    "uid": "ff72e5ce-02fe-4c3c-892d-cc1fcf9fdbce"
  },
  "security-labels": {
    "k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name": "kube-system",
    "k8s:io.cilium.k8s.policy.cluster": "default",
    "k8s:io.cilium.k8s.policy.serviceaccount": "coredns",
    "k8s:io.kubernetes.pod.namespace": "kube-system",
    "k8s:k8s-app": "kube-dns"
  }
}

kubectl exec -it -n kube-system ds/cilium -- cilium identity list
...
14735   k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=kube-system
        k8s:io.cilium.k8s.policy.cluster=default
        k8s:io.cilium.k8s.policy.serviceaccount=coredns
        k8s:io.kubernetes.pod.namespace=kube-system
        k8s:k8s-app=kube-dns
...

kubectl get pod -n kube-system -l k8s-app=kube-dns --show-labels
NAME                       READY   STATUS    RESTARTS   AGE   LABELS
coredns-674b8bbfcf-7v8gz   1/1     Running   0          14m   k8s-app=kube-dns,pod-template-hash=674b8bbfcf
coredns-674b8bbfcf-sbftb   1/1     Running   0          14m   k8s-app=kube-dns,pod-template-hash=674b8bbfcf
```
```bash
# 기동 중인 파드에 label 추가 후 cilium id(보안 lable)에 반영 될지? : 반영되는가? 얼마나 시간이 걸리는가? ID가 변경되지는 않는가?
kubectl label pods -n kube-system -l k8s-app=kube-dns study=8w
kubectl exec -it -n kube-system ds/cilium -- cilium identity list
...

# (위 내용 확인 후 아래 진행) coredns deployment 에 spec.template.metadata.labels에 아래 추가 시 반영 확인
kubectl edit deploy -n kube-system coredns
...
  template:
    metadata:
      labels:
        app: testing
        k8s-app: kube-dns
...

kubectl exec -it -n kube-system ds/cilium -- cilium identity list
...
```

- Cilium은 pod update 이벤트를 watch하므로, labels 변경 시 endpoint가 waiting-for-identity 상태로 전환되어 새로운 identity를 할당받습니다.
- 이로 인해 security labels와 관련된 네트워크 정책도 자동으로 재적용됩니다.
- 예시: 간단한 pod(simple-pod)를 생성하면 초기 security identity(예: 26830)가 할당됩니다.
- 이후 `kubectl label pod/simple-pod run=not-simple-pod --overwrite`로 labels를 변경하면, `kubectl get ciliumendpoints` 명령에서 identity가 새 값(예: 8710)으로 업데이트된 것을 확인할 수 있습니다. 이는 policy enforcement에 즉시 반영됩니다.
- 다만, 대규모 클러스터에서 자주 labels를 변경하면 identity 할당이 빈번해져 성능 저하가 발생할 수 있으므로, identity-relevant labels를 제한하는 것이 권장됩니다

- **Special Identities**
    - Cilium에서 관리하는 모든 엔드포인트에는 ID가 할당됩니다.
    - Cilium에서 관리하지 않는 네트워크 엔드포인트와의 통신을 허용하기 위해 이러한 엔드포인트를 나타내는 특수 ID가 존재합니다.
    - 특별히 예약된 ID에는 `reserved` 문자열 접두사가 붙습니다
```bash
kubectl exec -it -n kube-system ds/cilium -- cilium identity list
ID      LABELS
1       reserved:host
        reserved:kube-apiserver
2       reserved:world
3       reserved:unmanaged
4       reserved:health
5       reserved:init
6       reserved:remote-node
7       reserved:kube-apiserver
        reserved:remote-node
8       reserved:ingress
9       reserved:world-ipv4
10      reserved:world-ipv6
8836    k8s:app.kubernetes.io/instance=metrics-server
        k8s:app.kubernetes.io/name=metrics-server
        k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=kube-system
        k8s:io.cilium.k8s.policy.cluster=default
        k8s:io.cilium.k8s.policy.serviceaccount=metrics-server
...
```

|**Identity**|**Numeric ID**|**Description**|
|---|---|---|
|`reserved:unknown`|0|The identity could not be derived.|
|`reserved:host`|1|The local host. Any traffic that originates from or is designated to one of the local host IPs.|
|`reserved:world`|2|Any network endpoint outside of the cluster|
|`reserved:unmanaged`|3|An endpoint that is not managed by Cilium, e.g. a Kubernetes pod that was launched before Cilium was installed.|
|`reserved:health`|4|This is health checking traffic generated by Cilium agents.|
|`reserved:init`|5|An endpoint for which the identity has not yet been resolved is assigned the init identity. This represents the phase of an endpoint in which some of the metadata required to derive the security identity is still missing. This is typically the case in the bootstrapping phase.  <br>The init identity is only allocated if the labels of the endpoint are not known at creation time. This can be the case for the Docker plugin.|
|`reserved:remote-node`|6|The collection of all remote cluster hosts. Any traffic that originates from or is designated to one of the IPs of any host in any connected cluster other than the local node.|
|`reserved:kube-apiserver`|7|Remote node(s) which have backend(s) serving the kube-apiserver running.|
|`reserved:ingress`|8|Given to the IPs used as the source address for connections from Ingress proxies.|

* Identity Management in the Cluster
![[Pasted image 20250906222857.png]]

- ID는 전체 클러스터에서 유효합니다.
- 즉, 여러 클러스터 노드에서 여러 개의 포드 또는 컨테이너가 시작되더라도 ID 관련 레이블을 공유하는 경우 모든 포드 또는 컨테이너가 단일 ID를 확인하고 공유합니다.
- 이를 위해서는 클러스터 노드 간의 조정이 필요합니다.
- 엔드포인트 ID를 확인하는 작업은 **분산 키-값 저장소**를 통해 수행됩니다.
- 분산 키-값 저장소는 _다음 값이 이전에 확인되지 않은 경우 새로운 고유 식별자를 생성하는_ 형태의 원자적 연산을 수행할 수 있도록 합니다.
- 이를 통해 각 클러스터 노드는 ID 관련 레이블 하위 집합을 생성한 다음 키-값 저장소를 쿼리하여 ID를 도출할 수 있습니다.
- 레이블 집합이 이전에 쿼리되었는지 여부에 따라 새 ID가 생성되거나 초기 쿼리의 ID가 반환됩니다.


# Identity Management Mode - [Docs](https://docs.cilium.io/en/stable/network/kubernetes/identity-management-mode/)

# NetworkPolicy : 3가지 형식 제공 - [Docs](https://docs.cilium.io/en/stable/network/kubernetes/policy/)
![[Pasted image 20250906223252.png]]
- Pod의 유입 또는 유출 시 L3 및 L4 정책을 지원하는 **표준** [NetworkPolicy 리소스입니다.](https://docs.cilium.io/en/stable/network/kubernetes/policy/#networkpolicy)
- **3~7 계층**에서 **수신** 및 **송신** 모두에 대한 정책 지정을 지원하는 [CustomResourceDefinition](https://docs.cilium.io/en/stable/glossary/#term-CustomResourceDefinition) 으로 제공되는 확장된 [CiliumNetworkPolicy 형식입니다.](https://docs.cilium.io/en/stable/network/kubernetes/policy/#ciliumnetworkpolicy)
- [CiliumClusterwideNetworkPolicy](https://docs.cilium.io/en/stable/network/kubernetes/policy/#ciliumclusterwidenetworkpolicy) 형식 은 Cilium에서 적용할 **클러스터 전체 정책을 지정**하는 클러스터 범위 [CustomResourceDefinition](https://docs.cilium.io/en/stable/glossary/#term-CustomResourceDefinition) 입니다. 이 사양은 네임스페이스가 지정되지 않은 [CiliumNetworkPolicy](https://docs.cilium.io/en/stable/network/kubernetes/policy/#ciliumnetworkpolicy) 와 동일합니다 .

## Policy Enforcement Modes by Cilium Network Policy - [Docs](https://docs.cilium.io/en/stable/security/policy/intro/)
- three policy enforcement modes
    - default
    - always
    - never
- **Endpoint default policy**
    - **기본적으로 모든 엔드포인트에 대해 모든 송신 및 수신 트래픽이 허용됩니다. 네트워크 정책에 따라 엔드포인트가 선택되면 명시적으로 허용된** 트래픽 만 허용되는 기본 거부 상태로 전환됩니다 . 이 상태는 방향별로 적용됩니다.
        - 규칙이 [엔드포인트를](https://docs.cilium.io/en/stable/gettingstarted/terminology/#endpoint) 선택 하고 해당 규칙에 수신 섹션이 있는 경우, 엔드포인트는 수신에 대해 기본 거부 모드로 전환됩니다.
        - 규칙이 [엔드포인트를](https://docs.cilium.io/en/stable/gettingstarted/terminology/#endpoint) 선택 하고 규칙에 송신 섹션이 있는 경우, 엔드포인트는 송신에 대해 기본 거부 모드로 전환됩니다.
    - `EnableDefaultDeny` [7계층 정책](https://docs.cilium.io/en/stable/security/policy/language/#l7-policy) 에는 적용되지 않습니다.
        - 7계층 모두 허용이 포함되지 않은 7계층 규칙을 추가하면 default-deny가 명시적으로 비활성화된 경우에도 삭제가 발생합니다.
- **Rule Basics**
    - 엔드포인트 선택기 / 노드 선택기
        - 정책 규칙이 적용될 엔드포인트 또는 노드를 선택합니다. 정책 규칙은 선택기에 지정된 레이블과 일치하는 모든 엔드포인트에 적용.
    - 입구
        - 엔드포인트의 유입 시, 즉 엔드포인트에 들어오는 모든 네트워크 패킷에 적용해야 하는 규칙 목록
    - 출구
        - 엔드포인트의 출구에서 적용되어야 하는 규칙 목록, 즉 엔드포인트를 떠나는 모든 네트워크 패킷에 적용되어야 하는 규칙 목록
    - 라벨
        - [레이블은 규칙을 식별하는 데 사용됩니다. 레이블을 사용하여 규칙을 나열하고 삭제할 수 있습니다. Kubernetes를](https://docs.cilium.io/en/stable/network/kubernetes/policy/#k8s-policy) 통해 가져온 정책 규칙에는 [NetworkPolicy](https://docs.cilium.io/en/stable/network/kubernetes/policy/#networkpolicy) 또는 [CiliumNetworkPolicy](https://docs.cilium.io/en/stable/network/kubernetes/policy/#ciliumnetworkpolicy) 리소스 에 지정된 이름에 해당하는 레이블이 자동으로 `io.cilium.k8s.policy.name=NAME`지정됨.


# 정책 에디터 온라인 사이트 - [https://editor.networkpolicy.io/](https://editor.networkpolicy.io/) , [https://isovalent.com/blog/post/tutorial-network-policy-editor/](https://isovalent.com/blog/post/tutorial-network-policy-editor/)

# 도전과제1 Layer 3 예시 - [Docs](https://docs.cilium.io/en/stable/security/policy/language/)

^e1a84d

## [엔드포인트 기반](https://docs.cilium.io/en/stable/security/policy/language/#endpoints-based) : 두 엔드포인트가 Cilium에 의해 관리되고 레이블이 할당되는 경우 관계를 설명하는 데 사용됩니다. 이 방법의 장점은 IP 주소가 정책에 인코딩되지 않고 정책이 주소 지정과 완전히 분리된다는 것입니다.
```bash
# Simple Ingress 허용
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l3-rule"
spec:
  endpointSelector:
    matchLabels:
      role: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: frontend
```

## [서비스 기반](https://docs.cilium.io/en/stable/security/policy/language/#services-based) : 레이블과 CIDR의 중간 형태로, 오케스트레이션 시스템의 서비스 개념을 활용합니다. 쿠버네티스의 서비스 엔드포인트 개념이 좋은 예입니다. 이 엔드포인트는 서비스의 모든 백엔드 IP 주소를 포함하도록 자동으로 유지됩니다. 이를 통해 Cilium에서 대상 엔드포인트를 제어하지 않더라도 정책에 IP 주소를 하드코딩하지 않아도 됩니다.
```bash
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "service-rule"
spec:
  endpointSelector:
    matchLabels:
      id: app2
  egress:
  - toServices:
    # Services may be referenced by namespace + name
    - k8sService:
        serviceName: myservice
        namespace: default
    # Services may be referenced by namespace + label selector
    - k8sServiceSelector:
        selector:
          matchLabels:
            env: staging
        namespace: another-namespace
```

## [엔티티 기반](https://docs.cilium.io/en/stable/security/policy/language/#entities-based) : 엔티티는 IP 주소를 알지 못해도 분류할 수 있는 원격 피어를 설명하는 데 사용됩니다. 여기에는 엔드포인트를 제공하는 로컬 호스트에 대한 연결이나 클러스터 외부에 대한 모든 연결이 포함됩니다.

```bash
# 레이블이 있는 모든 엔드포인트가 env=devkube-apiserver에 액세스하도록 허용
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "dev-to-kube-apiserver"
spec:
  endpointSelector:
    matchLabels:
      env: dev
  egress:
    - toEntities:
      - kube-apiserver
```

## [노드 기반](https://docs.cilium.io/en/stable/security/policy/language/#node-based) : 엔티티의 확장입니다 remote-node . 선택적으로 노드는 고유한 ID를 가질 수 있으며, 이를 사용하여 특정 노드의 접근만 허용하거나 차단할 수 있습니다.
```bash
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "to-prod-from-control-plane-nodes"
spec:
  endpointSelector:
    matchLabels:
      env: prod
  ingress:
    - fromNodes:
        - matchLabels:
            node-role.kubernetes.io/control-plane: ""
```

## [IP/CIDR 기반](https://docs.cilium.io/en/stable/security/policy/language/#cidr-based) : 원격 피어가 엔드포인트가 아닌 경우 외부 서비스와의 관계를 설명하는 데 사용됩니다. 이를 위해서는 IP 주소 또는 서브넷을 정책에 직접 지정해야 합니다. 이 구성은 안정적인 IP 또는 서브넷 할당이 필요하므로 최후의 수단으로 사용해야 합니다.
```bash
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "cidr-rule"
spec:
  endpointSelector:
    matchLabels:
      app: myService
  egress:
  - toCIDR:
    - 20.1.1.1/32
  - toCIDRSet:
    - cidr: 10.0.0.0/8
      except:
      - 10.96.0.0/12
```

## [DNS 기반](https://docs.cilium.io/en/stable/security/policy/language/#dns-based) : DNS 조회를 통해 IP로 변환된 DNS 이름을 사용하여 원격의 비클러스터 피어를 선택합니다. [위 IP/CIDR 기반](https://docs.cilium.io/en/stable/security/policy/language/#cidr-based) 규칙의 모든 제한 사항을 공유합니다. DNS 정보는 프록시를 통해 DNS 트래픽을 라우팅하여 수집합니다. DNS TTL은 준수됩니다.

- DNS 정책은 Cilium에서 관리하지 않지만 DNS 쿼리 가능 도메인 이름을 가진 엔드포인트에 대한 계층 3 정책을 정의하는 데 사용됩니다.
- DNS 응답에 제공된 IP 주소는 [CIDR 기반](https://docs.cilium.io/en/stable/security/policy/language/#cidr-based) 정책의 IP와 유사한 방식으로 Cilium에서 허용됩니다
- 도메인 이름과 IP 주소를 연결하기 위해 Cilium은 [DNS 프록시를](https://docs.cilium.io/en/stable/security/policy/language/#dns-proxy) 사용하여 엔드포인트별 DNS 응답을 가로챕니다 .
- 이를 위해서는 Cilium이 DNS 요청을 허용하는 L7 정책으로 구성되어 있어야 합니다 `--enable-l7-proxy=true`.
- 모든 규칙에 대해 L3 [CIDR 기반](https://docs.cilium.io/en/stable/security/policy/language/#cidr-based) 규칙이 생성되어 `toFQDNs` 동일한 엔드포인트에 적용됩니다.
- IP 정보는 규칙에 의해 삽입되도록 선택되며 `matchName`, `matchPattern`노드에서 Cilium이 확인한 모든 DNS 응답에서 수집됩니다.
- DNS 프록시는 각 Cilium 에이전트에 제공됩니다. 따라서 정책의 대상이 되는 DNS 요청은 Cilium 에이전트 포드의 가용성에 따라 달라집니다.
- DNS 기반 규칙은 외부 연결을 위한 것이며 [CIDR 기반](https://docs.cilium.io/en/stable/security/policy/language/#cidr-based) 규칙과 유사하게 동작합니다.
- 클러스터 내부 트래픽에 대해서는 [서비스 기반](https://docs.cilium.io/en/stable/security/policy/language/#services-based) 및 [엔드포인트 기반을 참조하세요.](https://docs.cilium.io/en/stable/security/policy/language/#endpoints-based)
- 허용할 IP는 다음을 통해 선택됩니다.
- **`toFQDNs.matchName`**
    - 정확히 일치하는 도메인의 IP 주소를 삽입합니다 `matchName`.
    - 여러 개의 고유 이름을 별도의 `matchName`항목에 포함할 수 있으며, 일치하는 도메인의 IP 주소가 `matchName`삽입됩니다.
- **`toFQDNs.matchPattern`**
    - 와일드카드를 고려하여 패턴과 일치하는 도메인의 IP 주소를 삽입합니다 `matchPattern`. 패턴은 도메인 이름에 허용되는 문자(az, 0-9, .)로 구성 `.`됩니다 `-`.
    - 다음과 같은 여러 가지 편의 동작을 갖춘 와일드카드로 허용됩니다.
        - 도메인 내에서 `.`구분 기호를 제외하고 0개 이상의 유효한 DNS 문자를 허용합니다.
        - `.cilium.io`는 일치 `sub.cilium.io`하지만 `cilium.io`또는 는 일치하지 않습니다 `sub.sub.cilium.io`. 는 및 와 `part*ial.com`일치합니다 .`partial.compart-extra-ial.com`
        - 단독으로는 모든 이름과 일치하며, 캐시된 모든 DNS IP를 이 규칙에 삽입합니다.
```bash
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "to-fqdn"
spec:
  endpointSelector:
    matchLabels:
      app: test-app
  egress:
    - toEndpoints:
      - matchLabels:
          "k8s:io.kubernetes.pod.namespace": kube-system
          "k8s:k8s-app": kube-dns
      toPorts:
        - ports:
           - port: "53"
             protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
    - toFQDNs:
        - matchName: "my-remote-service.com"
```


# 도전과제2 Layer 4 예시 - [Docs](https://docs.cilium.io/en/stable/security/policy/language/#layer-4-examples)

^d9aa80

- 계층 4 정책은 계층 3 정책과 함께 또는 독립적으로 지정할 수 있습니다.
- 엔드포인트가 특정 프로토콜을 사용하여 특정 포트에서 패킷을 송수신하는 기능을 제한합니다.
- 엔드포인트에 계층 4 정책이 지정되지 않은 경우, 엔드포인트는 ICMP를 포함한 모든 계층 4 포트 및 프로토콜에서 송수신할 수 있습니다.
- 계층 4 정책이 지정된 경우, 정책에서 허용하는 연결과 관련된 경우를 제외하고는 차단됩니다.
- 계층 4 정책은 서비스 포트 매핑이 적용된 후 포트에 적용됩니다.
- 레이어 4 정책은 필드를 사용하여 수신 및 송신 모두에서 지정할 수 있습니다

* L4 정책 (예시)
```bash
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l4-rule"
spec:
  endpointSelector:
    matchLabels:
      app: myService
  egress:
    - toPorts:
      - ports:
        - port: "80"
          protocol: TCP
```

* 포트 범위 (예시) : 80~444 포트 범위
```bash
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l4-port-range-rule"
spec:
  endpointSelector:
    matchLabels:
      app: myService
  egress:
    - toPorts:
      - ports:
        - port: "80"
          endPort: 444
          protocol: TCP
```
* CIDR-dependent Layer 4 규칙 :
```bash
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "cidr-l4-rule"
spec:
  endpointSelector:
    matchLabels:
      role: crawler
  egress:
  - toCIDR:
    - 192.0.2.0/24
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

* TLS SNI 제한
	- 여러 웹사이트가 동일한 서버에 공유 IP 주소로 호스팅되는 경우, TLS 프로토콜의 확장 기능인 서버 이름 표시(SNI)는 클라이언트가 접속하려는 웹사이트에 대한 올바른 SSL 인증서를 수신하도록 보장합니다.
	- SNI를 사용하면 HTTP 연결이 설정될 때 핸드셰이크 이후가 아닌, TLS 핸드셰이크 중에 웹사이트의 호스트 이름 또는 도메인 이름을 지정할 수 있습니다.
	- Cilium 네트워크 정책은 엔드포인트가 TLS 핸드셰이크를 설정하는 기능을 지정된 SNI 목록으로 제한할 수 있습니다.
	- SNI 정책은 항상 송신 레벨에서 구성되며 일반적으로 포트 정책과 함께 설정됩니다.
	- TLS SNI 정책을 시행하려면 L7 프록시를 활성화해야 합니다.

* TLS SNI 정책 (예시)
```bash
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l4-sni-rule"
spec:
  endpointSelector:
    matchLabels:
      app: myService
  egress:
  - toPorts:
    - ports:
      - port: "443"
        protocol: TCP
      serverNames: # SNI
      - one.one.one.one
```

# 도전과제3 Layer 7 예시 - [Docs](https://docs.cilium.io/en/stable/security/policy/language/#layer-7-examples)

^d4e6e2

- 7계층 정책 규칙은 [4계층 예제](https://docs.cilium.io/en/stable/security/policy/language/#l4-policy) 규칙에 내장되어 있으며 수신 및 송신에 대해 지정할 수 있습니다.
- `L7Rules`구조는 프로토콜별 필드의 열거형을 포함하는 기본 유형입니다.
```bash
// L7Rules is a union of port level rule types. Mixing of different port
// level rule types is disallowed, so exactly one of the following must be set.
// If none are specified, then no additional port level rules are applied.
type L7Rules struct {
        // HTTP specific rules.
        //
        // +optional
        HTTP []PortRuleHTTP `json:"http,omitempty"`

        // Kafka-specific rules.
        //
        // +optional
        Kafka []PortRuleKafka `json:"kafka,omitempty"`

        // DNS-specific rules.
        //
        // +optional
        DNS []PortRuleDNS `json:"dns,omitempty"`
}
```
- 이 구조는 공용체로 구현됩니다. 즉, 포트당 하나의 멤버 필드만 사용할 수 있습니다.
- `toPorts`동일한 항목을 가진 여러 규칙이 `PortProtocol`중복되는 엔드포인트 목록을 선택하는 경우, 동일한 유형이면 계층 7 규칙이 결합됩니다.
- 유형이 다르면 정책이 거부됩니다.

- 각 멤버는 애플리케이션 프로토콜 규칙 목록으로 구성됩니다. 규칙 중 하나라도 일치하면 7계층 요청이 허용됩니다.
- 규칙이 지정되지 않으면 모든 트래픽이 허용됩니다.
- 정책에 계층 4 규칙이 지정되어 있고, 계층 7 규칙이 포함된 유사한 계층 4 규칙도 지정된 경우, 후자 규칙의 계층 7 부분은 아무런 효과가 없습니다.
- **3계층 및 4계층 정책과 달리 7계층 규칙 위반은 패킷 손실로 이어지지 않습니다**. 대신, 가능한 경우 **애플리케이션 프로토콜별 접근 거부 메시지**를 작성하여 반환합니다. 예를 들어, 정책을 위반하는 HTTP 요청에는 _HTTP 403 접근 거부_ 메시지를, DNS 요청에는 _DNS 거부_ 응답을 보냅니다.
- **레이어 7 정책**은 노드 로컬 [Envoy](https://docs.cilium.io/en/stable/security/network/proxy/envoy/#envoy) 인스턴스를 통해 **트래픽을 프록시**하며, 이 인스턴스는 DaemonSet으로 배포되거나 에이전트 포드에 내장됩니다.
- Envoy가 에이전트 포드에 내장된 경우, 정책이 적용되는 레이어 7 트래픽은 Cilium 에이전트 포드의 가용성에 따라 달라집니다.

## HTTP
- Path
    - Path는 요청 경로와 일치하는 확장된 POSIX 정규 표현식입니다. 현재 RFC 3986에 정의된 URL의 기존 "경로" 부분에서 허용되지 않는 문자를 포함할 수 있습니다. 경로는 .(마침표)로 시작해야 합니다 `/`. 생략하거나 비어 있는 경우 모든 경로가 허용됩니다.
- Method
    - 메서드는 요청의 메서드와 일치하는 확장된 POSIX 정규식입니다(예 `GET`: , `POST`, `PUT`, `PATCH`, `DELETE`, …). 생략되거나 비어 있으면 모든 메서드가 허용됩니다.
- Host
    - 호스트는 요청의 호스트 헤더와 일치하는 확장된 POSIX 정규식입니다(예: ) `foo.com`. 생략되거나 비어 있는 경우 호스트 헤더 값은 무시됩니다.
- Headers
    - 헤더는 요청에 반드시 포함되어야 하는 HTTP 헤더 목록입니다. 생략하거나 비어 있으면 헤더의 존재 여부와 관계없이 요청이 허용됩니다.
    - 헤더 값에 대해 좀 더 고급 헤더 매칭을 수행할 수도 있습니다. `HeaderMatches`는 반드시 존재해야 하며 주어진 값과 일치해야 하는 HTTP 헤더 목록입니다. 'Mismatch' 필드는 일치하는 헤더가 없을 때 수행할 작업을 지정하는 데 사용할 수 있습니다.
- **GET /public 허용**
```bash
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "Allow HTTP GET /public from env=prod to app=service"
  endpointSelector:
    matchLabels:
      app: service
  ingress:
  - fromEndpoints:
    - matchLabels:
        env: prod
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/public"
```
* 헤더가 설정된 경우 모든 GET /path1 및 PUT /path2
```bash
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l7-rule"
spec:
  endpointSelector:
    matchLabels:
      app: myService
  ingress:
  - toPorts:
    - ports:
      - port: '80'
        protocol: TCP
      rules:
        http:
        - method: GET
          path: "/path1$"
        - method: PUT
          path: "/path2$"
          headers:
          - 'X-My-Header: true'
```

## 카프카(베타) : Skip - [Docs](https://docs.cilium.io/en/stable/security/policy/language/#kafka-beta)

## DNS 정책 및 IP 검색 - [Docs](https://docs.cilium.io/en/stable/security/policy/language/#dns-policy-and-ip-discovery)
- 정책은 DNS 트래픽에 적용되어 특정 DNS 쿼리 이름 또는 이름 패턴을 허용하거나 허용하지 않을 수 있습니다(쿼리 유형과 같은 다른 DNS 필드는 고려되지 않음). 이 정책은 DNS 프록시를 통해 적용되며, L3 [DNS 기반](https://docs.cilium.io/en/stable/security/policy/language/#dns-based) `toFQDNs` 규칙을 채우는 데 사용되는 IP 주소를 수집하는 데에도 사용됩니다.
- 계층 7 DNS 정책은 다른 계층 3 규칙 없이도 적용할 수 있지만 계층 7 규칙(계층 3 및 4 구성 요소 포함)이 있으면 다른 트래픽이 차단됩니다.
- DNS 정책은 다음을 통해 적용될 수 있습니다.
    - **`matchName` :** 정확히 일치하는 도메인에 대한 쿼리를 허용합니다
    - **`matchPattern` :** 와일드카드를 고려하여 패턴과 일치하는 도메인에 대한 쿼리를 허용합니다.
```bash
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: "tofqdn-dns-visibility"
spec:
  endpointSelector:
    matchLabels:
      any:org: alliance
  egress:
  - toEndpoints:
    - matchLabels:
       "k8s:io.kubernetes.pod.namespace": kube-system
       "k8s:k8s-app": kube-dns
    toPorts:
      - ports:
         - port: "53"
           protocol: ANY
        rules:
          dns:
            - matchName: "cilium.io"
            - matchPattern: "*.cilium.io"
            - matchPattern: "*.api.cilium.io"

  - toFQDNs:
      - matchName: "cilium.io"
      - matchName: "sub.cilium.io"
      - matchName: "service1.api.cilium.io"
      - matchPattern: "special*service.api.cilium.io"
    toPorts:
      - ports:
         - port: "80"
           protocol: TCP
```
- 쿠버네티스에서 DNS 정책을 적용할 때, `service.namespace.svc.cluster.local`에 대한 쿼리는 명시적으로 허용되어야 합니다 .`matchPattern: *.*.svc.cluster.local.`
- 마찬가지로, FQDN을 완성하기 위해 DNS 검색 목록에 의존하는 쿼리는 전체가 허용되어야 합니다.
- 예를 들어, `servicename`로 성공하는 쿼리는 또는 `servicename.namespace.svc.cluster.local.`로 허용되어야 합니다 .

### Obtaining DNS Data for use by toFQDNs
- IP는 프록시를 통해 DNS 요청을 가로채서 얻습니다. 이러한 IP는 `toFQDN`규칙을 사용하여 선택할 수 있습니다.
- DNS 응답은 TTL을 준수하여 Cilium 에이전트 내에 캐시됩니다.
- DNS 프록시는 송신 DNS 트래픽을 가로채고 응답에서 확인된 IP 주소를 기록합니다.
    - 이 가로채기 자체는 DNS 요청을 제어하는 별도의 정책 규칙이므로 별도로 지정해야 합니다.
    - DNS 요청에 정책을 적용하고 DNS 프록시를 구성하는 방법에 대한 자세한 내용은 [레이어 7 예제를](https://docs.cilium.io/en/stable/security/policy/language/#layer-7-examples) 참조하세요 .
- 애플리케이션에 대한 가로채기된 DNS 응답의 IP만 Cilium 정책 규칙에서 허용됩니다.
    - 특정 도메인 이름에 대해 Cilium 인스턴스가 관리하는 모든 포드에 대한 응답의 IP는 정책에 따라 허용됩니다(TTL 준수).
    - 이를 통해 허용된 IP가 애플리케이션에 반환된 IP와 일치하도록 할 수 있습니다.
    - DNS 프록시는 와일드카드 L7 DNS `matchPattern`규칙에서 허용된 응답의 IP를 규칙에서 사용할 수 있도록 허용하는 유일한 방법입니다 `toFQDNs`
```bash
# 다음 예제는 DNS 요청을 차단하지 않고 가로채기 방식으로 DNS 데이터를 수집합니다. 
# sub.cilium.io 및 모든 하위 도메인에 대한 L3 연결을 cilium.io허용 sub.cilium.io 합니다 
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: "tofqdn-dns-visibility"
spec:
  endpointSelector:
    matchLabels:
      any:org: alliance
  egress:
  - toEndpoints:
    - matchLabels:
       "k8s:io.kubernetes.pod.namespace": kube-system
       "k8s:k8s-app": kube-dns
    toPorts:
      - ports:
         - port: "53"
           protocol: ANY
        rules:
          dns:
            - matchPattern: "*"
  - toFQDNs:
      - matchName: "cilium.io"
      - matchName: "sub.cilium.io"
      - matchPattern: "*.sub.cilium.io"
```

## Alpine/musl deployments and DNS Refused - [Docs](https://docs.cilium.io/en/stable/security/policy/language/#alpine-musl-deployments-and-dns-refused)

- Some common container images treat the DNS `Refused` response when the [DNS Proxy](https://docs.cilium.io/en/stable/security/policy/language/#dns-proxy) rejects a query as a more general failure. This stops traversal of the search list defined in `/etc/resolv.conf`. It is common for pods to search by appending `.svc.cluster.local.` to DNS queries. When this occurs, a lookup for `cilium.io` may first be attempted as `cilium.io.namespace.svc.cluster.local.` and rejected by the proxy. Instead of continuing and eventually attempting `cilium.io.` alone, the Pod treats the DNS lookup is treated as failed.
- This can be mitigated with the `--tofqdns-dns-reject-response-code` option. The default is **`refused`** but **`nameError`** can be selected, causing the proxy to return a NXDomain response to refused queries.
- A more pod-specific solution is to configure `ndots` appropriately for each Pod, via `dnsConfig`, so that the search list is not used for DNS lookups that do not need it. See the [Kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-config) for instructions.




# 도전과제4 Disk based Cilium Network Policies - [Docs](https://docs.cilium.io/en/stable/security/policy/language/#disk-based-cilium-network-policies)

^7281c6

# 도전과제5 Host Policies - [Docs](https://docs.cilium.io/en/stable/security/policy/language/#host-policies) & Host Firewall - [Blog](https://cilium.io/blog/2025/08/15/host-firewall-primer/)

^1d5997
![[Pasted image 20250906224123.png]]

# Using Kubernetes Constructs In Policy : 정책에서 쿠버네티스 리소스를 활용 - [Docs](https://docs.cilium.io/en/stable/security/policy/kubernetes/)

# Cilium Endpoint Lifecycle : - [Docs](https://docs.cilium.io/en/stable/security/policy/lifecycle/)
![[Pasted image 20250906224038.png]]

- `restoring`: The endpoint was started before Cilium started, and Cilium is restoring its networking configuration.
- `waiting-for-identity`: Cilium is allocating a unique identity for the endpoint.
- `waiting-to-regenerate`: The endpoint received an identity and is waiting for its networking configuration to be (re)generated.
- `regenerating`: The endpoint’s networking configuration is being (re)generated. This includes programming eBPF for that endpoint.
- `ready`: The endpoint’s networking configuration has been successfully (re)generated.
- `disconnecting`: The endpoint is being deleted.
- `disconnected`: The endpoint has been deleted.

# Network Policy Troubleshooting - [Docs](https://docs.cilium.io/en/stable/security/policy/troubleshooting/) & Caveats - [Docs](https://docs.cilium.io/en/stable/security/policy/caveats/)

# Threat Model - [Docs](https://docs.cilium.io/en/stable/security/threat-model/)

# Tutorial: Cilium Network Policy in Practice (Part 2) - [Blog](https://isovalent.com/blog/post/tutorial-cilium-network-policy/)
