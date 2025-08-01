* 소개 
	- Monitoring Datapath State는 datapath state에 대한 성찰을 제공하지만, 기본적으로 L3/L4 패킷 이벤트에 대한 가시성만 제공합니다.
	- L7 프로토콜 가시성을 원한다면 L7 Cilium Network Policies 을 사용할 수 있습니다(레이어 7 예제 참조).
	- L7 트래픽에 대한 가시성을 활성화하려면 L7 규칙을 지정하는 CiliumNetworkPolicy를 만드세요.
	- CiliumNetworkPolicy에서 L7 규칙과 일치하는 트래픽 흐름이 Cilium에 표시되므로 최종 사용자에게 노출될 수 있습니다.
	- L7 네트워크 정책은 가시성을 가능하게 할 뿐만 아니라 포드에 들어오고 나가는 트래픽을 제한한다는 점을 기억하는 것이 중요합니다.
- 실습
	- 다음 예제는 DNS(TCP/UDP/53) 및 HTTP(포트 TCP/80 및 TCP/8080) 트래픽을 기본 네임스페이스 내에서 표시할 수 있도록 두 가지 L7 규칙을 지정합니다.
	- 하나는 DNS 규칙이고 다른 하나는 HTTP 규칙입니다. 한 출력 통신을 제한하고 일치하지 않는 모든 것을 삭제합니다.
	- 규칙의 L7 일치 조건이 생략되거나 와일드카드 처리되어 각 규칙의 L4 섹션과 일치하는 모든 요청이 허용됩니다:
	- L7 네트워크 정책(가시성) 생성 및 확인
	    - 참고 : DNS visibility is available on egress only
```bash
# 반복 접속 해둔 상태
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s webpod | grep Hostname; sleep 1; done'


# default 네임스페이스에 있는 Pod들의 egress(출방향) 트래픽을 제어하며, L7 HTTP 및 DNS 트래픽에 대한 가시성과 제어를 설정
## method/path 기반 필터링은 안 하지만, HTTP 요청 정보는 Envoy를 통해 기록/관찰됨
## cilium-envoy를 경유하게 됨 (DNS + HTTP 모두 L7 처리 대상)
## 이 정책이 적용되면, 명시된 egress 외의 모든 egress 트래픽은 차단됩니다 (Cilium 정책은 default-deny 모델임)
cat <<EOF | kubectl apply -f -
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l7-visibility"
spec:
  endpointSelector:
    matchLabels:
      "k8s:io.kubernetes.pod.namespace": default  # default 네임스페이스 안의 모든 Pod에 대해 egress 정책이 적용
  egress:
  - toPorts:
    - ports:
      - port: "53"
        protocol: ANY  # TCP, UDP 둘 다 허용
      rules:
        dns:
        - matchPattern: "*"  # 모든 도메인 조회 허용, L7 가시성 활성화
  - toEndpoints:
    - matchLabels:
        "k8s:io.kubernetes.pod.namespace": default
    toPorts:
    - ports:
      - port: "80"  # default 다른 파드의 HTTP TCP 80 요청 허용
        protocol: TCP
      - port: "8080"  # default 다른 파드의 HTTP TCP 8080 요청 허용
        protocol: TCP
      rules:
        http: [{}]  # 모든 HTTP 요청을 허용, L7 가시성 활성화
EOF

kubectl get cnp -o yaml


# 호출 확인 : cilium-envoy 경유 확인
kubectl exec -it curl-pod -- curl -s webpod
Hostname: webpod-697b545f57-mt7vg
IP: 127.0.0.1
IP: ::1
IP: 172.20.2.141
IP: fe80::9840:7cff:feb3:a974
RemoteAddr: 172.20.0.48:37766  # 해당 IP는 curl-pod 의 IP로 cilium-envoy IP로 SNAT 되지 않았음!
GET / HTTP/1.1
Host: webpod
User-Agent: curl/8.14.1
Accept: */*
X-Envoy-Expected-Rq-Timeout-Ms: 3600000  # cilium-envoy 경유 확인
X-Envoy-Internal: true
X-Forwarded-Proto: http
X-Request-Id: 1c85d879-dc91-401e-b8a0-64ed80a92ffa


# 가시성 확인
hubble observe -f -t l7 -o compact
Jul 20 08:46:05.306: default/curl-pod:37700 (ID:20680) -> kube-system/coredns-674b8bbfcf-zz8pv:53 (ID:21239) dns-request proxy FORWARDED (DNS Query webpod.default.svc.cluster.local. A)
Jul 20 08:46:05.311: default/curl-pod:37700 (ID:20680) <- kube-system/coredns-674b8bbfcf-zz8pv:53 (ID:21239) dns-response proxy FORWARDED (DNS Answer "10.96.1.51" TTL: 30 (Proxy webpod.default.svc.cluster.local. A))
Jul 20 08:46:05.315: default/curl-pod:53570 (ID:20680) -> default/webpod-697b545f57-mt7vg:80 (ID:35772) http-request FORWARDED (HTTP/1.1 GET http://webpod/)
Jul 20 08:46:05.322: default/curl-pod:53570 (ID:20680) <- default/webpod-697b545f57-mt7vg:80 (ID:35772) http-response FORWARDED (HTTP/1.1 200 6ms (GET http://webpod/))
...
혹은 cilium nonitor ~
```
![[Pasted image 20250727124837.png]]


