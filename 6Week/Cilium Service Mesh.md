# 서비스 메시는 이런 공통 관심사(애플리케이션 네트워킹)를 애플리케이션 대신 외부에서 투명한 방식으로 구현하는 인프라
- 클라우드에서 마이크로서비스를 운영하는 데는 여러 도전과제가 있다.
    - 몇 가지를 꼽자면 신뢰할 수 없는 네트워크, 서비스 가용성, 이해하기 어려운 트래픽 흐름, 트래픽 암호화, 애플리케이션 상태, 성능 등이 있다.
- 이런 어려움들은 각 애플리케이션 내에서 라이브러리를 사용해 패턴(서비스 디스커버리 등)들을 구현함으로써 완화된다.
- 서비스들에 대한 관찰 가능성을 확보할 목적으로 메트릭과 트레이싱을 생성하고 배포하려면 추가적인 라이브러리와 서비스가 필요한다.
- **서비스 메시**는 이런 공통 관심사를 **애플리케이션 대신 프로세스 외부에서 투명한 방식으로 구현하는 인프라**다.
- 이스티오는 다음과 같은 요소로 구성된 서비스 메시 구현체다.
    - **데이터 플레인**, 애플리케이션과 함께 배포되는 서비스 프록시로 구성되며, 정책을 구현하고 트래픽을 제어하고 메트릭과 트레이싱을 생성하는 등 애플리케이션을 보완한다.
    - **컨트롤 플레인**, 운영자가 데이터 플레인의 네트워크 동작을 제어할 수 있도록 API를 노출한다.
- 이스티오는 **엔보이**를 서비스 프록시로 사용하는데, 기능이 다양하고 동적으로 설정할 수 있기 때문이다.

# 일반적인 서비스 매시(Service Mesh) 소개

등장 배경 : 마이크로서비스 아키텍처 환경의 시스템 전체 모니터링의 어려움, 운영 시 시스템 장애나 문제 발생할 때 원인과 병목 구간 찾기 어려움

내부망 진입점에 역할을 하는 GW(예. API Gateway) 경우 모든 동작 처리에 무거워지거나, 내부망 내부 통신 제어는 어려움

- **개념** : 마이크로서비스 간에 매시 형태의 **통신**이나 그 **경로**를 **제어** - 예) 이스티오(Istio), 링커드(Linkerd) - [링크](https://layer5.io/service-mesh-landscape)
- **기본 동작** : 파드 간 통신 경로에 프록시를 놓고 **트래픽 모니터링**이나 **트래픽 컨트롤** → 기존 애플리케이션 **코드에 수정 없이** 구성 가능!

## 기존 통신 환경
![[Pasted image 20250817210247.png]]
## 애플리케이션 수정 없이, 모든 애플리케이션 통신 사이에 Proxy 를 두고 통신 해보자
![[Pasted image 20250817210302.png]]
- 파드 내에 사이드카 컨테이너로 주입되어서 동작
- Proxy 컨테이너가 Application 트래픽을 가로채야됨 → **iptables rule** 구현 ⇒ 가능한 이유는?

## Proxy 는 결국 DataPlane 이니, 이를 중앙에서 관리하는 ControlPlane을 두고 중앙에서 관리를 하자
![[Pasted image 20250817210337.png]]
- Proxy 는 중앙에서 설정 관리가 잘되는 툴을 선택. 즉, 원격에서 동적인 설정 관리가 유연해야함 → 풍부한 API 지원이 필요 ⇒ Envoy
    - '구글 IBM 리프트(Lyft)'가 중심이 되어 개발하고 있는 오픈 소스 소프트웨어이며, C++ 로 구현된 **고성능 Proxy** 인 엔보이(Envoy)
    - 네트워크의 투명성을 목표, 다양한 **필터체인** 지원(L3/L4, HTTP L7), **동적 configuration API 제공, api 기반 hot reload 제공**
- 중앙에서 어떤 동작/설정을 관리해야 될까? 라우팅, 보안 통신을 위한 mTLS 관련, 동기화 상태 정보 등


- **트래픽 모니터링** : 요청의 '에러율, 레이턴시, 커넥션 개수, 요청 개수' 등 메트릭 모니터링, 특정 서비스간 혹은 특정 요청 경로로 필터링 → 원인 파악 용이!
- **트래픽 컨트롤** : 트래픽 시프팅(Traffic shifting), 서킷 브레이커(Circuit Breaker), 폴트 인젝션(Fault Injection), 속도 제한(Rate Limit)
    - 트래픽 시프팅(Traffic shifting) : 예시) 99% 기존앱 + 1% 신규앱 , 특정 단말/사용자는 신규앱에 전달하여 단계적으로 적용하는 카니리 배포 가능
    - 서킷 브레이커(Circuit Breaker) : 목적지 마이크로서비스에 문제가 있을 시 접속을 차단하고 출발지 마이크로서비스에 요청 에러를 반환 (연쇄 장애, 시스템 전제 장애 예방)
    - 폴트 인젝션(Fault Injection) : 의도적으로 요청을 지연 혹은 실패를 구현
    - 속도 제한(Rate Limit) : 요청 개수를 제한

# Cilium Service Mesh 소개* - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/) , [Youtube](https://www.youtube.com/watch?v=lZskwr3uXn8)

![[Pasted image 20250817210841.png]]
- **Why Cilium Service Mesh?**
    - **L3/L4 수준 프로토콜 : eBPF 처리**
        - **IP, TCP, UD**P와 같은 프로토콜을 포함한 모든 네트워크 처리에 대해 Cilium은 고효율 커널 내부 데이터 경로로 **eBPF**를 사용.
        - _For all network processing including protocols such as **IP**, **TCP**, and **UDP**, Cilium uses **eBPF** as the **highly efficient in-kernel datapath**_
	- **L7 수준 애플리케이션 : cilium-envoy 처리**
	    - HTTP, Kafka, gRPC, DNS와 같은 **애플리케이션** 계층 프로토콜은 **Envoy**와 같은 프록시를 사용하여 파싱.
	    - _Protocols at the **application** **layer** such as HTTP, Kafka, gRPC, and DNS are parsed using a **proxy** such as **Envoy**._

- 제공 기능
    - **Resilient Connectivity**: Service to service communication must be possible across boundaries such as clouds, clusters, and premises. Communication must be **resilient** and **fault tolerant.**
    - **L7 Traffic Management**: Load balancing, rate limiting, and resiliency must be L7-aware (HTTP, REST, gRPC, WebSocket, …).
    - **Identity-based Security**: Relying on network identifiers to achieve security is no longer sufficient, both the sending and receiving services must be able to authenticate each other based on identities instead of a network identifier.
    - **Observability & Tracing**: Observability in the form of tracing and metrics is critical to understanding, monitoring, and troubleshooting application stability, performance, and availability.
    - **Transparency**: The functionality must be available to applications in a transparent manner, i.e. without requiring to change application code.

