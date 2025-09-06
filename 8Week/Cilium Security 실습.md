# [Lab1] Locking Down External Access with DNS-Based Policies : DNS 기반 보안 정책 - [Docs](https://docs.cilium.io/en/stable/security/dns/)

- CIDR 또는 IP 기반 정책은 외부 서비스와 연결된 IP가 자주 변경될 수 있으므로 관리가 어렵고 번거롭습니다.
- Cilium의 DNS 기반 정책은 DNS-IP 매핑 추적과 같은 복잡한 측면을 관리하는 동시에 액세스 제어를 쉽게 지정할 수 있는 메커니즘을 제공합니다.
- 이 가이드에서는 다음 사항에 대해 알아봅니다.
    - DNS 기반 정책을 사용하여 클러스터 외부 서비스에 대한 이탈 액세스 제어
    - 패턴(또는 와일드카드)을 사용하여 DNS 도메인 하위 집합을 허용 목록에 추가
    - 외부 서비스 접근 제한을 위한 DNS, 포트 및 L7 규칙 결합

## 데모 애플리케이션 배포
- Empire의 mediabot pod가 Empire의 git 저장소 관리를 위해 GitHub에 접근해야 하는 간단한 시나리오를 사용하겠습니다.
- pod는 다른 외부 서비스에 접근할 수 없어야 합니다.
```bash
# hubble ui 에 pod name 표기를 위해 app labels 추가 >> 빼고 배포해보고 차이점을 확인해보자.
cat << EOF > dns-sw-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mediabot
  labels:
    org: empire
    class: mediabot
    app: mediabot
spec:
  containers:
  - name: mediabot
    image: quay.io/cilium/json-mock:v1.3.8@sha256:5aad04835eda9025fe4561ad31be77fd55309af8158ca8663a72f6abb78c2603
EOF
```

```bash
kubectl apply -f dns-sw-app.yaml
kubectl wait pod/mediabot --for=condition=Ready

# 확인
kubectl exec -it -n kube-system ds/cilium -- cilium identity list
kubectl get pods
NAME       READY   STATUS    RESTARTS   AGE
mediabot   1/1     Running   0          41s

# 외부 통신 확인 : hubble ui 에서 확인
kubectl exec mediabot -- curl -I -s https://api.github.com | head -1
kubectl exec mediabot -- curl -I -s --max-time 5 https://support.github.com | head -1

```
## DNS Egress 정책 적용  1 : mediabot포드가 api.github.com에만 액세스하도록 허용
```bash
#
cat << EOF | kubectl apply -f -
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "fqdn"
spec:
  endpointSelector:
    matchLabels:
      org: empire
      class: mediabot
  egress:
  - toFQDNs:
    - matchName: "api.github.com"
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
EOF
```
```bash

# 확인
kubectl get cnp
NAME   AGE   VALID
fqdn   4s    True

kubectl exec -it -n kube-system ds/cilium -- cilium policy selectors 
SELECTOR                                                                                                                                                                      LABELS         USERS   IDENTITIES
&LabelSelector{MatchLabels:map[string]string{any.class: mediabot,any.org: empire,k8s.io.kubernetes.pod.namespace: default,},MatchExpressions:[]LabelSelectorRequirement{},}   default/fqdn   1       5190 

cilium config view | grep -i dns
dnsproxy-enable-transparent-mode                  true
dnsproxy-socket-linger-timeout                    10
hubble-metrics                                    dns drop tcp flow port-distribution icmp httpV2:exemplars=true;labelsContext=source_ip,source_namespace,source_workload,destination_ip,destination_namespace,destination_workload,traffic_direction
tofqdns-dns-reject-response-code                  refused
tofqdns-enable-dns-compression                    true
tofqdns-endpoint-max-ip-per-hostname              1000
tofqdns-idle-connection-grace-period              0s
tofqdns-max-deferred-connection-deletes           10000
tofqdns-preallocate-identities                    true
tofqdns-proxy-response-max-delay                  100ms


# 외부 통신 확인 : hubble ui 에서 확인
kubectl exec mediabot -- curl -I -s https://api.github.com | head -1

kubectl exec mediabot -- curl -I -s --max-time 5 https://support.github.com | head -1

# cilium-agent 내에 go 로 구현된 lightweight proxy 가 DNS 쿼리/응답 감시와 캐싱 처리 : 기본 30초?
cilium hubble port-forward&
hubble observe --pod mediabot
Aug 31 03:59:53.209: default/mediabot:47883 (ID:5190) <- kube-system/coredns-674b8bbfcf-p6pbn:53 (ID:1363) dns-response proxy FORWARDED (DNS Answer "20.200.245.245" TTL: 30 (Proxy api.github.com. A))
...
Aug 31 03:59:53.212: default/mediabot:38212 (ID:5190) -> api.github.com:443 (ID:16777217) policy-verdict:L3-Only EGRESS ALLOWED (TCP Flags: SYN)
...

# (옵션) coredns 로그로 직접 확인해보기
k9s -> configmap (coredns) : log 추가

## 로깅 활성화
kubectl logs -n kube-system -l k8s-app=kube-dns -f

## 호출
kubectl exec mediabot -- curl -I -s https://api.github.com | head -1
kubectl exec mediabot -- curl -I -s https://api.github.com | head -1
hubble observe --pod mediabot

## cilium 파드 이름
export CILIUMPOD0=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-ctr -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD1=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w1  -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD2=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w2  -o jsonpath='{.items[0].metadata.name}')
echo $CILIUMPOD0 $CILIUMPOD1 $CILIUMPOD2

## 단축키(alias) 지정
alias c0="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- cilium"
alias c1="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- cilium"
alias c2="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- cilium"

##
c0 fqdn cache list
c1 fqdn cache list
c2 fqdn cache list
Endpoint   Source       FQDN                  TTL   ExpirationTime             IPs               
363        connection   support.github.com.   0     2025-08-31T05:10:05.726Z   185.199.111.133   
363        connection   support.github.com.   0     2025-08-31T05:10:05.726Z   185.199.108.133   
363        connection   api.github.com.       0     2025-08-31T05:10:05.726Z   20.200.245.245   

c0 fqdn names
c1 fqdn names
c2 fqdn names
{
  "DNSPollNames": null,
  "FQDNPolicySelectors": [
    {
      "regexString": "^api[.]github[.]com[.]$",
      "selectorString": "MatchName: api.github.com, MatchPattern: "
    }
  ]
}
```
* cilium-agent 내에 go 로 구현된 lightweight proxy 가 DNS 쿼리/응답 감시와 캐싱 처리 - [Code](https://github.com/cilium/cilium/blob/main/pkg/fqdn/dnsproxy/proxy.go)
![[Pasted image 20250906224354.png]]

## DNS Egress 정책 적용 2 : 모든 GitHub 하위 도메인(예: 패턴)에 액세스.
```bash
# fqdn 캐시 초기화 및 정책 삭제
kubectl delete cnp fqdn
c1 fqdn cache clean -f
c2 fqdn cache clean -f


# dns-pattern.yaml 내용
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "fqdn"
spec:
  endpointSelector:
    matchLabels:
      org: empire
      class: mediabot
  egress:
  - toFQDNs:
    - matchName: "*.github.com"
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


kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.18.1/examples/kubernetes-dns/dns-pattern.yaml
c1 fqdn names
c2 fqdn names
c1 fqdn cache list
c2 fqdn cache list


# 확인
kubectl get cnp
NAME   AGE   VALID
fqdn   4s    True


# 외부 통신 확인 : hubble ui 에서 확인 >> github.com 은 공식 문서 설명대로라면 안되야됨..
##  It is important to note and test that this doesn’t allow access to github.com because the *. 
## in the pattern requires one subdomain to be present in the DNS name
kubectl exec mediabot -- curl -I -s https://support.github.com | head -1

kubectl exec mediabot -- curl -I -s https://gist.github.com | head -1

kubectl exec mediabot -- curl -I -s --max-time 5 https://github.com | head -1

kubectl exec mediabot -- curl -I -s --max-time 5 https://cilium.io| head -1
```
![[Pasted image 20250906224436.png]]

## DNS Egress 정책 적용 3 : DNS, Port 조합 적용

```bash
# dns-port.yaml 내용
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "fqdn"
spec:
  endpointSelector:
    matchLabels:
      org: empire
      class: mediabot
  egress:
  - toFQDNs:
    - matchPattern: "*.github.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
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


kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.18.1/examples/kubernetes-dns/dns-port.yaml
c1 fqdn names
c2 fqdn names
c1 fqdn cache list
c2 fqdn cache list
c1 fqdn cache clean -f
c2 fqdn cache clean -f


# 외부 통신 확인 : hubble ui 에서 확인 
kubectl exec mediabot -- curl -I -s https://support.github.com | head -1

kubectl exec mediabot -- curl -I -s --max-time 5 http://support.github.com | head -1
```

## 실습 리소스 삭제
```bash
#
kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/1.18.1/examples/kubernetes-dns/dns-sw-app.yaml
kubectl delete cnp fqdn
```

# 샘플 애플리케이션 배포 및 통신 문제 확인
## 샘플 애플리케이션 배포
```bash 
# 샘플 애플리케이션 배포
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webpod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webpod
  template:
    metadata:
      labels:
        app: webpod
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - sample-app
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: webpod
        image: traefik/whoami
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: webpod
  labels:
    app: webpod
spec:
  selector:
    app: webpod
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
EOF


# k8s-ctr 노드에 curl-pod 파드 배포
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: curl-pod
  labels:
    app: curl
spec:
  nodeName: k8s-ctr
  containers:
  - name: curl
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
EOF
```

## 통신 확인
```bash
# 배포 확인
kubectl get deploy,svc,ep webpod -owide
kubectl get endpointslices -l app=webpod
kubectl get ciliumendpoints # IP 확인

# 통신 문제 확인
kubectl exec -it curl-pod -- curl -s --connect-timeout 1 webpod | grep Hostname
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s --connect-timeout 1 webpod | grep Hostname; echo "---" ; sleep 1; done'

# cilium-dbg, map
kubectl exec -n kube-system ds/cilium -- cilium-dbg ip list
kubectl exec -n kube-system ds/cilium -- cilium-dbg endpoint list
kubectl exec -n kube-system ds/cilium -- cilium-dbg service list
kubectl exec -n kube-system ds/cilium -- cilium-dbg bpf lb list
kubectl exec -n kube-system ds/cilium -- cilium-dbg bpf nat list
kubectl exec -n kube-system ds/cilium -- cilium-dbg map list | grep -v '0             0'
kubectl exec -n kube-system ds/cilium -- cilium-dbg map get cilium_lb4_services_v2
kubectl exec -n kube-system ds/cilium -- cilium-dbg map get cilium_lb4_backends_v3
kubectl exec -n kube-system ds/cilium -- cilium-dbg map get cilium_lb4_reverse_nat
kubectl exec -n kube-system ds/cilium -- cilium-dbg map get cilium_ipcache_v2

```

# Transparent Encryption with WireGuard 소개
- Each node automatically creates its own encryption key-pair and distributes its public key via the `io.cilium.network.wg-pub-key` **annotation** in the Kubernetes **CiliumNode** custom resource object.
- Each node’s **public key** is then used by other nodes to decrypt and encrypt traffic from and to **Cilium-managed endpoints** running on that node.
- 한 노드 내에서 파드(엔드포인트)간 통신 시에는 암호화 되지 않습니다.
- The WireGuard tunnel endpoint is exposed on **UDP** port **51871** on each node.
- Limitations : L7 policy enforcement and visibility , eBPF-based host routing

![[Pasted image 20250906224547.png]]

![[Pasted image 20250906224607.png]]![[Pasted image 20250906224622.png]]

# [Lab2] WireGuard 설정 및 실습 : 터널 모드는 두 번 캡슐화됨 - [Docs](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/) , [Youtube](https://www.youtube.com/watch?v=-awkPi3D60E)

- WireGuard 터널 엔드포인트는 `51871`각 노드의 UDP 포트에 노출.
- 설정
```bash
# [커널 구성 옵션] CONFIG_WIREGUARD=m on Linux 5.6 and newe 
uname -ar
grep -E 'CONFIG_WIREGUARD=m' /boot/config-$(uname -r)

# 설정 전 기본 정보 확인
ip -c addr
ip -c route
ip rule show

# 설정
helm upgrade cilium cilium/cilium --version 1.18.1 --namespace kube-system --reuse-values \
  --set encryption.enabled=true --set encryption.type=wireguard

kubectl -n kube-system rollout restart ds/cilium

# 확인
cilium config view | grep -i wireguard
kubectl exec -it -n kube-system ds/cilium -- cilium encrypt status
Encryption: Wireguard                 
Interface: cilium_wg0
        Public key: o4ISkG+hN88144j0FCwTEo0NtfvLnID3/q48eJ1pfVQ=
        Number of peers: 2

kubectl exec -it -n kube-system ds/cilium -- cilium status | grep Encryption
Encryption:              Wireguard   [NodeEncryption: Disabled, cilium_wg0 (Pubkey: o4ISkG+hN88144j0FCwTEo0NtfvLnID3/q48eJ1pfVQ=, Port: 51871, Peers: 2)]   

kubectl exec -it -n kube-system ds/cilium -- cilium debuginfo --output json
kubectl exec -it -n kube-system ds/cilium -- cilium debuginfo --output json | jq .encryption

ip -d -c addr show cilium_wg0
ip rule show


# wg 정보 확인
wg -h
wg show
interface: cilium_wg0
  public key: UywNVxgvRBwecznbmGk/vHcD1gTXtmCi9kEQEZd5lxc=
  private key: (hidden)
  listening port: 51871
  fwmark: 0xe00

peer: jm4w0DGrufZ5yINKr327f6aG4AAgqI+yy19cCtpCGnA=
  endpoint: 192.168.10.102:51871
  allowed ips: 192.168.10.102/32, 172.20.2.0/24, 172.20.2.197/32, 172.20.2.82/32, 172.20.2.219/32, 172.20.2.2/32
  latest handshake: 2 minutes, 31 seconds ago
  transfer: 556 B received, 468 B sent

peer: o4ISkG+hN88144j0FCwTEo0NtfvLnID3/q48eJ1pfVQ=
  endpoint: 192.168.10.101:51871
  allowed ips: 172.20.1.60/32, 192.168.10.101/32, 172.20.1.97/32, 172.20.1.0/24, 172.20.1.118/32

wg show all public-key
wg show all private-key
wg show all preshared-keys
wg show all endpoints
wg show all transfer

# 퍼블릭 키 확인
kubectl get cn -o yaml | grep annotations -A1

```

## 통신 확인
```bash
# curl 호출
kubectl exec -it curl-pod -- curl webpod
kubectl exec -it curl-pod -- curl webpod


# tcpdump
tcpdump -i cilium_wg0 -n
tcpdump -eni any udp port 51871
tcpdump -eni any udp port 51871 -w /tmp/wg.pcap  # vagrant scp k8s-ctr:/tmp/wg.pcap . > wireshark 로 확인

# query the flow API and look for flows
hubble observe --pod curl-pod
```

![[Pasted image 20250906224847.png]]

## Node-to-Node Encryption (beta) : 노드 - 노드 , 노드 - 파드 간 트래픽도 암호 --set encryption.nodeEncryption=true

## 설정 원복
```bash
#
helm upgrade cilium cilium/cilium --version 1.18.1 --namespace kube-system --reuse-values \
  --set encryption.enabled=false

kubectl -n kube-system rollout restart ds/cilium


# 확인
cilium config view | grep -i wireguard
kubectl exec -it -n kube-system ds/cilium -- cilium encrypt status
kubectl exec -it -n kube-system ds/cilium -- cilium status | grep Encryption
```


# Inspecting TLS Encrypted Connections with Cilium - [Docs](https://docs.cilium.io/en/stable/security/tls-visibility/)
![[Pasted image 20250906224927.png]]
- **소개**
    
    - 이 문서는 네트워크 보안 팀이 Cilium을 사용하여 **TLS 암호화 연결을 투명하게 검사**하는 방법을 소개합니다.
    - 이 TLS 인식 검사는 클라이언트가 HTTPS를 통해 API 서비스에 액세스하는 경우처럼 **클라이언트 간 통신이 TLS로 보호되는 연결에서도 Cilium API 인식 가시성 및 정책을 작동**시킬 수 있도록 합니다.
    - 이 기능은 기존 하드웨어 방화벽과 유사하지만, Kubernetes 워커 노드에서 전적으로 소프트웨어로 구현되며 정책 기반으로 작동하므로 선택된 네트워크 연결만 검사할 수 있습니다.
    - 네트워크 보안 모니터링을 위한 "친근한 차단"의 경우, Cilium은 TLS 검사 기능을 갖춘 기존 방화벽과 유사한 모델을 사용합니다.
    - 네트워크 보안 팀은 외부 대상에 대한 대체 인증서를 생성하는 데 사용할 수 있는 **자체 "내부 인증 기관"을 생성**합니다.
    - 이 모델에서는 각 클라이언트 워크로드가 이 새 인증서를 신뢰해야 하며, 그렇지 않으면 클라이언트의 TLS 라이브러리가 연결을 유효하지 않은 것으로 간주하여 거부합니다.
    - 이 모델에서 네트워크 방화벽은 내부 CA에서 서명한 인증서를 사용하여 대상 서비스처럼 작동하고 TLS 연결을 종료합니다.
    - 이를 통해 방화벽은 애플리케이션 계층 데이터를 검사하고 수정한 후 실제 대상 서비스에 대한 다른 TLS 연결을 시작할 수 있습니다.
- TLS의 CA 모델은 암호화 키와 인증서를 기반으로 합니다. 위 모델을 구현하려면 **네 가지 주요 단계**가 필요
    
    1. CA 개인 키와 CA 인증서를 생성하여 **내부 인증 기관**을 만듭니다.
    2. TLS 검사가 필요한 모든 대상(예: 아래 예의 [httpbin.org](http://httpbin.org))에 대해 대상 DNS 이름과 일치하는 일반 이름을 사용하여 **개인 키와 인증서 서명 요청**을 생성합니다.
    3. **CA 개인 키**를 사용하여 **서명된 인증서**를 만듭니다.
    4. TLS 검사를 실시하는 모든 클라이언트에 **CA 인증서가 설치**되어 해당 CA가 서명한 모든 인증서를 **신뢰**할 수 있는지 확인하세요.
    5. **Cilium**은 클라이언트와의 **초기 TLS 연결을 종**료하고 **대상에 대한 새로운 TLS 연결을 생성**하므로, 대상 서비스에 대한 새로운 TLS 연결을 검증할 때 Cilium이 신뢰해야 하는 **CA 집합**에 Cilium에 대한 정보를 알려야 합니다.
    
    <aside> 👉🏻
    
    데모 환경이 아닌 경우, 위의 개인 키를 안전하게 보관하는 것이 매우 중요합니다. 이 개인 키에 액세스할 수 있는 사람은 누구나 TLS 암호화 트래픽을 검사할 수 있기 때문입니다(반면 인증서는 공개 정보이므로 전혀 민감하지 않습니다). 아래 가이드에서 CA 개인 키는 Cilium에 제공할 필요가 없습니다(인증서 생성에만 사용되며, 오프라인에서 수행 가능). 개별 대상 서비스의 개인 키는 Kubernetes 비밀로 저장됩니다. 이러한 비밀은 Cilium에서 액세스할 수 있지만 범용 워크로드에서는 액세스할 수 없는 네임스페이스에 저장해야 합니다.
    
    </aside>
    
- **TLS 검사 작동 방식**
    
    - 모든 TLS 검사는 수락될 인증서를 사용하여 원래 연결을 종료한 다음, 필요한 경우 클라이언트 인증서를 사용하여 새로운 TLS 연결을 시작하는 데 의존합니다.
    - 이러한 이유로 네트워크 정책에서는 `terminatingTLS`및 선택적으로 `originatingTLS`스탠자를 구성해야 합니다.
    - 네트워크 정책에 이러한 세부 정보가 포함되어 있으면 **Cilium은 TLS 연결을 Envoy로 리디렉션**하고 TLS 핸드셰이크를 완료하고 구성된 네트워크 정책을 전달하는 연결을 허용합니다.
    - 이를 구성하는 데 가장 중요한 부분 중 하나는 **인증서를 Envoy에 전달**하는 방법입니다.
    - 현재 버전에서 Cilium에는 **NPDS**(원래 버전)와 **SDS**(새롭고 향상된 버전)의 두 가지 옵션이 있습니다.
    - **네트워크 정책 검색 서비스(NPDS)**
        - 이 버전에서는 인증서와 키가 Cilium 소유의 네트워크 정책 검색 서비스의 전용 필드에 Base64 **인코딩된 텍스트로 인라인으로 전송**됩니다.
        - 이 방법은 간단하게 구축할 수 있다는 장점이 있지만 **큰 단점도** 있습니다.
        - TLS 가로채기를 수행하는 각 네트워크 정책 규칙은 Envoy의 NPDS 구성에 각 비밀의 사본을 인라인으로 보관합니다. 따라서 (대규모 설치 환경에서 흔히 발생하는 것처럼) 동일한 비밀을 여러 번 재사용하는 경우(예: 여러 SAN에 대해 만료되는 인증서를 하나 생성했지만 해당 인증서를 사용하는 규칙이 여러 개 있거나 구성에 유효한 루트 인증서 번들을 포함하는 경우 `originatingTLS`) **인증서의 여러 사본이 Envoy의 메모리**에 저장됩니다. 이러한 **메모리 사용량은 대규모 설치 환경에서는 상당히 증가**할 수 있습니다.
    - **비밀 발견 서비스(SDS)**
        - 위의 두 가지 이유 때문에 Envoy는 **Cilium 1.17부터 네트워크 정책 비밀에 대한 SDS**를 지원합니다.
        - 이 구성에서 Cilium은 구성된 **시크릿 네임스페이스**에서 관련 시크릿을 읽고, 핵심 **Envoy SDS API를 사용하여 Envoy에 노출합**니다. 이러한 시크릿은 Envoy로 전송된 N**PDS 구성에서 참조되**며, Base64로 인코딩된 텍스트로 직접 포함되는 대신 이름을 기준으로 네트워크 정책 필터를 구성합니다.
        - 즉, Envoy는 **Ingress** 또는 **Gateway API 구성**에 대한 비밀을 찾는 것과 같은 방식으로 **NPDS에 대한 SDS 비밀을 찾습**니다.
        - 이 방법을 사용하면 Envoy가 비**밀의 저장을 중복 제거**할 수 있습니다. 비밀은 본질적으로 값으로 전달되는 것이 아니라 **참조로 전달**되기 때문입니다.
        - 이러한 NPDS 방식에 비해 우수한 점 때문에 Cilium 1.17부터는 새로운 **Cilium 설치에 SDS가 기본값**으로 적용됩니다.


- **Configuring TLS Interception** : There are three ways to use Cilium in 1.17 and later
    
    - **(권장) SDS**를 사용하면 네트워크 정책에서 참조되는 비밀을 클러스터 내 어디에나 배치할 수 있으며, `cilium-secrets`Cilium Operator가 이를 구성된 네임스페이스(기본값)에 복사하고, 해당 네임스페이스에서 SDS로 동기화한 후, 해당 이름을 사용하여 NPDS에서 참조합니다. 이는 새 클러스터의 기본 설정이며 권장되는 운영 방식입니다.
    - 시크릿은 클러스터 내 어디에나 위치할 수 있으며, Cilium 에이전트는 클러스터 내 모든 시크릿에 대한 읽기 권한을 부여받을 수 있습니다. 이 경우, 시크릿은 Cilium 에이전트가 원래 위치에서 직접 읽어 **NPDS**(네트워크 데이터 서비스)를 통해 **인라인**으로 전송됩니다. 이 배포 방식은 이전 버전과의 호환성을 위해 포함된 것으로, 에이전트 의 보안 범위를 **크게 확장하므로 권장되지 않습니다 .**
    - 비밀은 네임스페이스에 직접 추가된 `cilium-secrets`후 네트워크 정책에서 해당 네임스페이스를 참조할 수 있습니다. 이 기능은 이 기능이 실제로 어떻게 사용되었는지에 대한 사용자 피드백을 기반으로 하위 호환성을 유지하기 위해 포함되었습니다. 설정을 구성하지 않고 Helm의 설정을 사용하거나 그 이하로 설정된 **업그레이드된** 클러스터의 기본값입니다 .`upgradeCompatibility1.16`
    - **Helm** 관련 설정
        - `tls.secretsNamespace.nam` : 기본값 `cilium-secrets` : 정책 비밀에 사용될 비밀 네임스페이스를 구성합니다.
            - 이 값은 기본적으로 Ingress, Gateway API 및 BGP 구성의 유사한 설정과 동일한 값으로 설정되지만, 다른 값으로 설정될 **수 있습니다 .**
        - `tls.readSecretsOnlyFromSecretsNamespace` : 기본값 `true`.
            - 이 설정은 Helm 차트와 Cilium에 Cilium 에이전트가 구성된 Secrets 네임스페이스에서만 비밀을 읽어야 하는지, 아니면 Cilium 에이전트가 클러스터의 해당 위치에서 직접 비밀을 읽어야 하는지 알려줍니다. 이전 버전의 Cilium에서는 항목을 사용했는데 , 이 항목은 (Secrets 네임스페이스에서만 읽음) 또는 (모든 네임스페이스에서 읽음) `tls.secretsBackend`으로 설정할 수 있었지만 , 이름이 해당 기능과 분리되어 해당 필드는 더 이상 사용되지 **않습니다** . 로 설정된 이전 설치는 대신 로 설정해야 하지만, 이 설정은 Cilium 1.17에서도 계속 작동합니다. 향후 Cilium 버전에서 제거될 예정입니다.`localk8stls.secretsBackendk8stls.readSecretsOnlyFromSecretsNamespacefalsetls.secretsBackend`
        - `tls.secretSync.enabledtrue` : 새 클러스터의 기본값입니다.
            - 네트워크 정책 비밀에 대한 비밀 동기화 및 SDS 사용을 구성합니다.
            - SDS를 사용하려면 이 값을 로 설정해야 하며 `true`, 이 필드를 로 설정하면 비활성화해야 하므로 `false`SDS 구성을 위한 추가 필드가 있어도 값이 추가되지 않습니다.
- **Configuring the three available modes for TLS Interception**
    - **SDS Mode (recommended, default for new clusters):**
```bash
tls:
  readSecretsOnlyFromSecretsNamespace: true
  secretsNamespace:
    name: cilium-secrets # This setting is optional, as it is the default
  secretSync:
    enabled: true
```
	* Read all Secrets in the Cluster mode (not recommended)
```bash
tls:
  readSecretsOnlyFromSecretsNamespace: false
  secretSync:
    enabled: false
```
	* Read Secrets only from secrets namespace, no SDS (default for upgraded clusters)
```bash
tls:
  readSecretsOnlyFromSecretsNamespace: true
  secretsNamespace:
    name: cilium-secrets # This setting is optional, as it is the default
  secretSync:
    enabled: false
```

# Cilium TLS Interception 설정 (SDS Mode)
```bash 
#
kubectl get all,secret,cm -n cilium-secrets
NAME                         DATA   AGE
configmap/kube-root-ca.crt   1      7h31m

#
cat << EOF > tls-config.yaml
tls:
  readSecretsOnlyFromSecretsNamespace: true
  secretsNamespace:
    name: cilium-secrets # This setting is optional, as it is the default
  secretSync:
    enabled: true
EOF

helm upgrade cilium cilium/cilium --version 1.18.1 --namespace kube-system --reuse-values \
-f tls-config.yaml

kubectl -n kube-system rollout restart deploy/cilium-operator
kubectl -n kube-system rollout restart ds/cilium

# 확인
cilium config view | grep -i secret
enable-ingress-secrets-sync                       true
enable-policy-secrets-sync                        true
ingress-secrets-namespace                         cilium-secrets
policy-secrets-namespace                          cilium-secrets
policy-secrets-only-from-secrets-namespace        true

```

# [Lab3] Cilium TLS Interception Demo 실습 - [Docs](https://docs.cilium.io/en/stable/security/tls-visibility/#deploy-the-demo-application)

- TLS 가로채기를 시연하기 위해 DNS 인식 정책 예제에 사용한 것과 동일한 `mediabot`애플리케이션을 사용할 것입니다. 이 애플리케이션은 HTTPS를 사용하여 Star Wars API 서비스에 액세스합니다. 이는 일반적으로 Cilium과 같은 네트워크 계층 메커니즘이 통신의 HTTP 계층 세부 정보를 볼 수 없음을 의미합니다. 모든 애플리케이션 데이터는 네트워크로 전송되기 전에 TLS를 사용하여 암호화되기 때문입니다.
- 이 가이드에서는 다음 사항에 대해 알아봅니다.
    - **TLS 가로채기를 활성화**하기 위해 **내부 인증 기관(CA)과 해당 CA가 서명한 관련 인증서**를 만듭니다.
    - **DNS 기반 정책 규칙**을 사용하여 **가로챌 트래픽을 선택**하려면 **Cilium 네트워크 정책을** 사용합니다.
    - cilium monitor를 사용하여 H**TTP 요청의 세부 사항을 검사**합니다
        - (Hubble을 통해 이 가시성 데이터에 액세스하고 Cilium 네트워크 정책을 적용하여 HTTP 요청을 필터링/수정하는 것도 가능하지만 이 간단한 시작 가이드의 범위를 벗어납니다)
- 단일 포드 mediabot 애플리케이션을 생성

```bash 
# 
cat << EOF > dns-sw-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mediabot
  labels:
    org: empire
    class: mediabot
    app: mediabot
spec:
  containers:
  - name: mediabot
    image: quay.io/cilium/json-mock:v1.3.8@sha256:5aad04835eda9025fe4561ad31be77fd55309af8158ca8663a72f6abb78c2603
EOF
```
```bash
kubectl apply -f dns-sw-app.yaml
kubectl wait pod/mediabot --for=condition=Ready

# 확인
kubectl get pods
kubectl exec mediabot -- curl -I -s https://api.github.com | head -1
kubectl exec mediabot -- curl -I -s --max-time 5 https://support.github.com | head -1

```

## TLS 키 및 인증서 생성 및 설치
![[Pasted image 20250906225254.png]]
- 이제 TLS를 이해하고 Cilium을 구성하여 TLS 가로채기를 사용했으므로 유틸리티를 사용하여 적절한 키와 인증서를 생성하는 구체적인 단계를 살펴보겠습니다.
- 다음 이미지는 생성되거나 복사되는 암호화 데이터가 포함된 다양한 파일과 시스템의 어떤 구성 요소가 해당 파일에 액세스해야 하는지 설명합니다.
- 로컬 시스템에 openssl이 이미 설치되어 있다면 사용할 수 있지만, 그렇지 않은 경우 간단한 단축키를 사용하여 cilium 포드 내에서 실행한 후 결과 명령을 실행할 수 있습니다.
- Kubernetes 시크릿을 생성하거나 포드에 복사할 때 cilium 포드에서 결과 파일을 복사하는 데 사용할 수 있습니다 .

## 내부 인증 기관(CA) 만들기
```bash
# 'myCA.key'라는 이름의 CA 개인 키를 생성합니다.
openssl genrsa -des3 -out myCA.key 2048
Enter PEM pass phrase: qwe123  # 아무 비밀번호나 입력하세요. 이후 단계에서 사용할 수 있도록 기억.
Verifying - Enter PEM pass phrase: qwe123

ls *.key
myCA.key

# 개인 키에서 CA 인증서를 생성
openssl req -x509 -new -nodes -key myCA.key -sha256 -days 1825 -out myCA.crt
Enter pass phrase for myCA.key: qwe123 
Country Name (2 letter code) [AU]:KR
State or Province Name (full name) [Some-State]:Seoul
Locality Name (eg, city) []: Seoul
Organization Name (eg, company) [Internet Widgits Pty Ltd]: cloudneta
Organizational Unit Name (eg, section) []:study
Common Name (e.g. server FQDN or YOUR name) []:cloudneta.net
Email Address []:

ls *.crt
myCA.crt

openssl x509 -in myCA.crt -noout -text
        Issuer: C = KR, ST = Seoul, L = Seoul, O = cloudneta, OU = study, CN = cloudneta.net
        Validity
            Not Before: Aug 31 08:31:07 2025 GMT
            Not After : Aug 30 08:31:07 2030 GMT
        Subject: C = KR, ST = Seoul, L = Seoul, O = cloudneta, OU = study, CN = cloudneta.net
        ...
            X509v3 Basic Constraints: critical
                CA:TRUE
```

## 지정된 DNS 이름에 대한 개인 키 및 인증서 서명 요청 생성
```bash
# 검사를 위해 가로채려는 대상 서비스의 DNS 이름과 일치하는 일반 이름을 사용하여 내부 개인 키와 인증서 서명을 생성합니다
## 이 예에서는 httpbin.org

# 먼저 개인 키를 만듭니다.
openssl genrsa -out internal-httpbin.key 2048
ls internal-httpbin.key

# 다음으로, 인증서 서명 요청을 생성하고, 메시지가 표시되면 일반 이름 필드에 대상 서비스의 DNS 이름을 지정합니다. 
# 다른 모든 메시지에는 원하는 값을 입력할 수 있습니다.
# 특정 값이어야 하는 유일한 필드는 클라이언트에 제공될 정확한 DNS 대상을 보장하는 것입니다. Common Name: httpbin.org
# 이 예제 워크플로는 정책 YAML(아래)의 toFQDNs 규칙도 인증서의 DNS 이름과 일치하도록 업데이트되는 한 모든 DNS 이름에 적용됩니다.
openssl req -new -key internal-httpbin.key -out internal-httpbin.csr
Common Name (e.g. server FQDN or YOUR name) []:httpbin.org

ls internal-httpbin.csr
```

## CA를 사용하여 DNS 이름에 대한 서명된 인증서 생성
```bash
# 내부 CA 개인 키를 사용하여 httpbin.org에 대한 서명된 인증서를 생성합니다 internal-httpbin.crt
openssl x509 -req -days 360 -in internal-httpbin.csr -CA myCA.crt -CAkey myCA.key -CAcreateserial -out internal-httpbin.crt -sha256
Certificate request self-signature ok
subject=C = KR, ST = Seoul, L = Seoul, O = cloudneta, OU = study, CN = httpbin.org
Enter pass phrase for myCA.key: qwe123

ls internal-httpbin.crt
openssl x509 -in internal-httpbin.crt -noout -text
...
        Issuer: C = KR, ST = Seoul, L = Seoul, O = cloudneta, OU = study, CN = cloudneta.net
        Validity
            Not Before: Aug 31 08:36:33 2025 GMT
            Not After : Aug 26 08:36:33 2026 GMT
        Subject: C = KR, ST = Seoul, L = Seoul, O = cloudneta, OU = study, CN = httpbin.org

# 다음으로 대상 서비스에 대한 개인 키와 서명된 인증서를 모두 포함하는 Kubernetes 비밀을 생성합니다.
kubectl create secret tls httpbin-tls-data -n kube-system --cert=internal-httpbin.crt --key=internal-httpbin.key
kubectl get secret -n kube-system  httpbin-tls-data

```

## 클라이언트 포드 내에서 신뢰할 수 있는 CA로 내부 CA 추가
* CA 인증서가 클라이언트 포드에 저장된 후에도 애플리케이션에서 사용하는 TLS 라이브러리가 CA 파일을 인식하는지 확인해야 합니다. 대부분의 Linux 애플리케이션은 Linux 배포판과 함께 제공되는 신뢰할 수 있는 CA 인증서 세트를 자동으로 사용합니다. 이 가이드에서는 Ubuntu 컨테이너를 클라이언트로 사용하므로 Ubuntu별 지침에 따라 업데이트합니다. 다른 Linux 배포판은 다른 메커니즘을 사용합니다. 또한, 개별 애플리케이션은 OS 인증서 저장소를 사용하는 대신 자체 인증서 저장소를 활용할 수 있습니다. Java 애플리케이션과 aws-cli가 대표적인 예입니다. 자세한 내용은 애플리케이션 또는 애플리케이션 런타임 설명서를 참조하십시오.

```bash
#
kubectl exec -it mediabot -- ls -l /usr/local/share/ca-certificates/

# Ubuntu의 경우 먼저 추가 CA 인증서를 클라이언트 Pod 파일 시스템에 복사합니다.
kubectl cp myCA.crt default/mediabot:/usr/local/share/ca-certificates/myCA.crt
kubectl exec -it mediabot -- ls -l /usr/local/share/ca-certificates/

# /etc/ssl/certs/ca-certificates.crt에 있는 신뢰할 수 있는 인증 기관의 글로벌 세트에 이 인증서를 추가하는 Ubuntu 전용 유틸리티 실행
kubectl exec -it mediabot -- ls -l /etc/ssl/certs/ca-certificates.crt # 사이즈, 생성/수정 날짜 확인
kubectl exec mediabot -- update-ca-certificates
kubectl exec -it mediabot -- ls -l /etc/ssl/certs/ca-certificates.crt # 사이즈, 생성/수정 날짜 확인

```

- **Cilium에 신뢰할 수 있는 CA 목록 제공**
    - 다음으로, Cilium이 **보조 TLS 연결을 시작할** 때 **신뢰해야 하는 CA 세트를 제공**합니다. 이 목록은 조직에서 신뢰하는 **표준 글로벌 CA 세트와** 일치해야 합니다. 논리적인 옵션 중 하나는 운영 체제에서 신뢰하는 표준 CA 세트입니다. 이는 TLS 검사 도입 이전에 사용되던 CA 세트이기 때문입니다.
    - 간단하게 설명하기 위해 이 예에서 우리는 m**ediabot 포드의 Ubuntu 파일 시스템에서 이 목록을 복사**할 것입니다. 하지만 이 신뢰할 수 있는 CA 목록은 특정 TLS 클라이언트나 서버에만 국한되지 않는다는 점을 이해하는 것이 중요합니다. 따라서 TLS 검사에 참여하는 TLS 클라이언트나 서버의 수에 관계없이 이 단계는 한 번만 수행하면 됩니다.
```bash
#
kubectl cp default/mediabot:/etc/ssl/certs/ca-certificates.crt ca-certificates.crt

#  이 인증서 번들을 사용하여 Kubernetes 비밀을 생성하여 Cilium이 인증서 번들을 읽고 이를 사용하여 나가는 TLS 연결을 검증할 수 있도록 합니다.
kubectl create secret generic tls-orig-data -n kube-system --from-file=ca.crt=./ca-certificates.crt
kubectl get secret -n kube-system tls-orig-data

```

- **Apply DNS and TLS-aware Egress Policy**
    - 지금까지 TLS 검사를 활성화하기 위한 키와 인증서를 생성했지만, Cilium에 어떤 트래픽을 가로채서 검사할지 알려주지 않았습니다. 이 작업은 다른 Cilium 네트워크 정책에 사용되는 것과 동일한 Cilium 네트워크 정책 구조를 사용하여 수행됩니다.
    - 다음 Cilium 네트워크 정책은 Cilium이 `mediabot`포드와 . 사이의 통신에 대해 HTTP 인식 검사를 수행해야 함을 나타냅니다 `httpbin.org`.
```bash
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l7-visibility-tls"
spec:
  description: L7 policy with TLS
  endpointSelector:
    matchLabels:
      org: empire
      class: mediabot
  egress:
  - toFQDNs:
    - matchName: "httpbin.org"
    toPorts:
    - ports:
      - port: "443"
        protocol: "TCP"
      terminatingTLS:
        secret:
          namespace: "kube-system"
          name: "httpbin-tls-data"
      originatingTLS:
        secret:
          namespace: "kube-system"
          name: "tls-orig-data"
      rules:
        http:
        - {}
  - toPorts:
    - ports:
      - port: "53"
        protocol: ANY
      rules:
        dns:
          - matchPattern: "*"
```

- 즉 , 이 정책은 출구 접근 권한이 있는 `endpointSelector`라벨이 있는 포드에만 적용됩니다 .`class: mediabot, org:empire`
- 첫 번째 출구 섹션에서는 사양을 사용하여 TCP 포트 443 출구를 허용합니다 .`toFQDNs: matchNamehttpbin.org`
- `http`toFQDNs 규칙 아래의 섹션은 이러한 연결이 모든 요청을 허용하는 정책에 따라 HTTP로 구문 분석되어야 함을 나타 냅니다 `{}`.
- 및 섹션은 **TLS 가로채기를 사용하여 mediabot에서의 초기 TLS 연결을 종료**하고 .에 대한 **새로운 아웃바운드 TLS 연결을 시작**해야 함을 나타 `terminatingTLS`냅니다 .`originatingTLShttpbin.org`
- 두 번째 출구 섹션은 `mediabot`포드가 `kube-dns`서비스에 접근할 수 있도록 합니다. Cilium이 지정된 패턴과 일치하는 DNS 조회를 검사하고 허용하도록 지시합니다. 이 경우 모든 DNS 쿼리를 검사하고 허용합니다.`rules: dns`

```bash
# 모니터링 : hubble cli 나 ui 에서 https 이니 L7 정보 확인 불가능
hubble observe --pod mediabot -f

# 
kubectl get pods
kubectl exec -it mediabot -- curl -sL 'https://httpbin.org/anything'
kubectl exec -it mediabot -- curl -sL 'https://httpbin.org/headers' -v # 서버 인증서 확인
* Server certificate:
*  subject: CN=httpbin.org
*  start date: Jul 20 00:00:00 2025 GMT
*  expire date: Aug 17 23:59:59 2026 GMT
*  subjectAltName: host "httpbin.org" matched cert's "httpbin.org"
*  issuer: C=US; O=Amazon; CN=Amazon RSA 2048 M03
*  SSL certificate verify ok.


# 정책 적용
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.18.1/examples/kubernetes-tls-inspection/l7-visibility-tls.yaml
kubectl get cnp

```

- **Demonstrating TLS Inspection**
    - 우리가 푸시한 정책은 에서 `mediabot`로의 모든 HTTPS 요청을 허용 `httpbin.org`하지만 모든 데이터는 HTTP 계층에서 구문 분석한다는 점을 기억하세요. 즉, cilium monitor는 모든 HTTP 요청과 응답을 보고합니다.
    - 이를 확인하려면 새 창을 열고 다음 명령을 실행하여 `mediabot`포드와 동일한 Kubernetes 워커 노드에서 실행 중인 cilium 포드의 이름(예: cilium-97s78)을 식별합니다.
```bash
# 모니터링 : hubble cli 나 ui 에서 https 이니 L7 정보 확인 불가능
hubble observe --pod mediabot -f

#
kubectl exec -it mediabot -- curl -sL 'https://httpbin.org/anything'
kubectl exec -it mediabot -- curl -sL 'https://httpbin.org/headers' -v # 서버 인증서 확인
* Server certificate:
*  subject: C=KR; ST=Seoul; L=Seoul; O=cloudneta; OU=study; CN=httpbin.org
*  start date: Aug 31 08:36:33 2025 GMT
*  expire date: Aug 26 08:36:33 2026 GMT
*  common name: httpbin.org (matched)
*  issuer: C=KR; ST=Seoul; L=Seoul; O=cloudneta; OU=study; CN=cloudneta.net
*  SSL certificate verify ok.

```

![[Pasted image 20250906225530.png]]

## 실습 리소스 삭제
```bash
kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/1.18.1/examples/kubernetes-dns/dns-sw-app.yaml
kubectl delete cnp l7-visibility-tls
kubectl delete secret -n kube-system tls-orig-data
kubectl delete secret -n kube-system httpbin-tls-data
```

# `도전과제6` Securing a Kafka Cluster - [Docs](https://docs.cilium.io/en/stable/security/kafka/)

^2ce029

# `도전과제7` Securing Elasticsearch - [Docs](https://docs.cilium.io/en/stable/security/elasticsearch/)

^5f072f

# `도전과제8` Securing gRPC - [Docs](https://docs.cilium.io/en/stable/security/grpc/)

^b1a882

# `도전과제9` Host Firewall - [Docs](https://docs.cilium.io/en/stable/security/host-firewall/)

^a9e518
