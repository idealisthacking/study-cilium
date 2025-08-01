# Encapsulation : VXLAN, GENEVE

^12740c

Encapsulation : VXLAN, GENEVE - [Docs](https://docs.cilium.io/en/stable/network/concepts/routing/)

- Cilium 기본 네트워킹 인프라에서 요구 사항이 가장 적은 모드이기 때문에 Encapsulation 모드에서 자동으로 실행됩니다.
- 이 모드에서는 모든 클러스터 노드가 **UDP** 기반 캡슐화 프로토콜 **VXLAN** 또는 **Geneve**를 사용하여 **터널**의 메시를 형성합니다.
- Cilium 노드 간의 모든 트래픽이 캡슐화됩니다.
- 캡슐화는 일반 노드 간 연결에 의존합니다. 이는 Cilium 노드가 이미 서로 연결될 수 있다면 모든 라우팅 요구 사항이 이미 충족된다는 것을 의미합니다.
- 기본 네트워크는 IPv4를 지원해야 합니다. 기본 네트워크와 방화벽은 캡슐화된 패킷을 허용해야 합니다:
    - VXLAN (default) : UDP 8472
    - Geneve : UDP 6081
- 장점
    - 단순성 Simplicity
        - 클러스터 노드를 연결하는 네트워크는 PodCIDR을 인식할 필요가 없습니다.
        - 클러스터 노드는 여러 라우팅 또는 링크 계층 도메인을 생성할 수 있습니다.
        - 클러스터 노드가 IP/UDP를 사용하여 서로 연결할 수 있는 한 기본 네트워크의 토폴로지는 중요하지 않습니다.
    - 정체성 맥락 Identity context
        - 캡슐화 프로토콜은 네트워크 패킷과 함께 메타데이터를 전송할 수 있게 해줍니다.
        - Cilium은 소스 보안 ID와 같은 메타데이터를 전송하는 이 기능을 활용합니다.
        - The identity transfer is an optimization designed to avoid one identity lookup on the remote node.
- 단점
    - **MTU Overhead**
        - 캡슐화 헤더를 추가하면 페이로드에 사용할 수 있는 유효 MTU가 네이티브 라우팅(VXLAN의 경우 네트워크 패킷당 50바이트)보다 낮아집니다.
        - 이로 인해 특정 네트워크 연결에 대한 최대 처리량이 낮아집니다.
        - 이는 점보 프레임(1500바이트당 오버헤드 50바이트 대 9000바이트당 오버헤드 50바이트)을 활성화함으로써 크게 완화될 수 있습니다.
- 설정
    - `tunnel-protocol`: Set the encapsulation protocol to `vxlan` or `geneve`, defaults to `vxlan`.
    - `tunnel-port`: Set the port for the encapsulation protocol. Defaults to `8472` for `vxlan` and `6081` for `geneve`.

# Native-Routing : PodCIDR

^a7118d

![[스크린샷 2025-07-28 오전 12.05.03.png]]
- 네이티브 라우팅 데이터 경로는 라우팅 모드에서 네이티브로 활성화되며 네이티브 패킷 전달 모드를 활성화합니다.
- 네이티브 패킷 전달 모드는 캡슐화를 수행하는 대신 Cilium이 실행되는 네트워크의 라우팅 기능을 활용합니다.
- 네이티브 라우팅 모드에서는 Cilium이 다른 로컬 엔드포인트로 주소 지정되지 않은 모든 패킷을 Linux 커널의 라우팅 하위 시스템에 위임합니다.
- 이는 패킷이 로컬 프로세스가 패킷을 방출한 것처럼 라우팅된다는 것을 의미합니다.
- 따라서 클러스터 노드를 연결하는 네트워크는 PodCIDR을 라우팅할 수 있어야 합니다.
- PodCIDR 라우팅 방안 1
    - 각 개별 노드는 다른 모든 노드의 모든 포드 IP를 인식하고 이를 표현하기 위해 Linux 커널 라우팅 테이블에 삽입됩니다.
    - 모든 노드가 단일 L2 네트워크를 공유하는 경우 `auto-direct-node-routes: true`하여 이 문제를 해결할 수 있습니다.
    - 그렇지 않으면 **BGP** 데몬과 같은 추가 시스템 구성 요소를 실행하여 경로를 배포해야 합니다.
- PodCIDR 라우팅 방안 2
    - 노드 자체는 모든 포드 IP를 라우팅하는 방법을 모르지만 다른 모든 포드에 도달하는 방법을 아는 라우터가 네트워크에 존재합니다.
    - 이 시나리오에서는 Linux 노드가 이러한 라우터를 가리키는 기본 경로를 포함하도록 구성됩니다.
    - 이 모델은 클라우드 제공자 네트워크 통합에 사용됩니다. 자세한 내용은 [Google Cloud](https://docs.cilium.io/en/stable/network/concepts/routing/#google-cloud), [AWS ENI](https://docs.cilium.io/en/stable/network/concepts/routing/#aws-eni) 및 Azure IPAM을 참조하세요.
- 설정
    - `routing-mode: native`: Enable native routing mode.
    - `ipv4-native-routing-cidr: x.x.x.x/y`: Set the CIDR in which native routing can be performed.
    - `auto-direct-node-routes: true` : 동일 L2 네트워크 공유 시, 걱 노드의 PodCIDR에 대한 Linux 커널 라우팅 테이블에 삽입.

# 노드 간 파드 통신 상세 확인 with Native-Routing
```bash
#
kubectl get pod -owide

# Webpod1,2 파드 IP
export WEBPODIP1=$(kubectl get -l app=webpod pods --field-selector spec.nodeName=k8s-ctr -o jsonpath='{.items[0].status.podIP}')
export WEBPODIP2=$(kubectl get -l app=webpod pods --field-selector spec.nodeName=k8s-w1  -o jsonpath='{.items[0].status.podIP}')
echo $WEBPODIP1 $WEBPODIP2

# curl-pod 에서 WEBPODIP2 로 ping
kubectl exec -it curl-pod -- ping $WEBPODIP2

# 커널 라우팅 확인
ip -c route
sshpass -p 'vagrant' ssh vagrant@k8s-w1 ip -c route


#
cilium hubble port-forward&
hubble observe -f --pod curl-pod
Jul 27 04:21:27.648: default/curl-pod (ID:15584) -> default/webpod-857687f8b6-wdxhq (ID:46396) to-network FORWARDED (ICMPv4 EchoRequest)
Jul 27 04:21:27.649: default/curl-pod (ID:15584) <- default/webpod-857687f8b6-wdxhq (ID:46396) to-endpoint FORWARDED (ICMPv4 EchoReply)

#
tcpdump -i eth1 icmp

#
tcpdump -i eth1 icmp -w /tmp/icmp.pcap
termshark -r /tmp/icmp.pcap

```
![[Pasted image 20250728001102.png]]


