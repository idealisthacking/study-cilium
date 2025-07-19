# Cilium과 eBPF: 차세대 네트워킹의 핵심 기술 🚀

[eBPF](https://ebpf.io/) 또는 [Cilium Docs](https://docs.cilium.io/en/stable/overview/intro/)

컨테이너와 클라우드 네이티브 환경이 대세가 되면서, 기존 네트워킹 기술들의 한계가 드러나기 시작함. 수천 개의 컨테이너를 연결하고, 마이크로서비스 간 통신을 관리하는 것은 쉽지 않은 일. 이런 문제들을 해결하기 위해 등장한 게 바로 **eBPF**와 **Cilium**임.

## eBPF가 뭐길래? 🤔

eBPF(Extended Berkeley Packet Filter)는 리눅스 커널 안에서 안전하게 프로그램을 실행할 수 있게 해주는 기술. 쉽게 말하면 커널을 건드리지 않고도 커널 레벨에서 원하는 작업을 할 수 있게 해주는 마법 같은 기술임. (XDP를 건드리게되면 Module을 Import하는 관점에서 커널을 건드리냐 안건드리냐가 좀 애매모호하긴하지만..)
![[Pasted image 20250719205228.png]]

### 옛날 BPF의 아픈 과거 😅

1992년에 나온 원조 BPF는 네트워크 패킷을 걸러내는 용도로만 사용됨. 기능이 너무 제한적이었음:

- 명령어가 몇 개 없어서 복잡한 건 못 함
- 패킷 필터링만 가능
- 확장성? 그게 뭔가요?

### eBPF의 등장, 게임 체인저 🎮

2014년 Alexei Starovoitov이 eBPF를 만들면서 상황이 완전히 바뀜:

- **명령어 대폭 확장**: 64비트 레지스터, 다양한 연산 지원
- **어디든 갈 수 있음**: 네트워킹, 시스템 호출, 트레이싱 등 커널 곳곳에서 실행
- **안전함**: 커널 크래시 걱정 없이 사용 가능
- **빠름**: JIT 컴파일로 네이티브 속도

## eBPF 왜 좋은데? 장단점 정리 ⚖️

### 👍 장점들

**안전성 최고**

- 커널 모듈처럼 시스템 다운시킬 걱정 없음
- 무한루프나 메모리 오류 자동으로 차단

**성능 미쳤음**

- 커널에서 바로 실행되니까 오버헤드 거의 없음
- JIT 컴파일로 최적화까지

**유연성 좋음**

- 커널 재컴파일 없이 실시간으로 프로그램 바꿀 수 있음
- 원하는 대로 커스텀 가능

**실시간 모니터링**

- 시스템 상태를 실시간으로 볼 수 있음
- 디버깅이나 성능 분석에 최적

### 👎 단점들

**러닝커브 가파름**

- 배우기 어려움... 특히 처음엔
- 디버깅도 쉽지 않음

**플랫폼 의존적**

- 리눅스 커널 버전따라 기능이 다름
- 업데이트할 때 호환성 체크 필수

**제약사항 있음**

- 스택 크기 제한, 명령어 수 제한 등
- 일부 커널 함수는 접근 못 함

## Cilium은 또 뭐야? 🐝

Cilium은 eBPF를 기반으로 만든 네트워킹, 보안, 관측성 솔루션. 특히 쿠버네티스 환경에서 진가를 발휘함.

### 왜 만들어졌을까? 🤷‍♂️

기존 네트워킹 솔루션들이 클라우드 네이티브 환경에서 한계를 보였음:

- **확장성 문제**: 컨테이너 수천 개 처리하기 벅참
- **성능 병목**: 여러 네트워크 레이어 거치면서 느려짐
- **보안 복잡**: IP 기반으론 한계가 있음
- **가시성 부족**: 마이크로서비스끼리 뭐 하는지 모르겠음

### Cilium의 킹왕짱 기능들 🎯

**API 레벨 보안**

- HTTP, gRPC, Kafka 같은 프로토콜까지 이해함
- L7 방화벽으로 세밀한 제어 가능

**성능 최적화**

- eBPF 덕분에 커널 바이패스로 빠름
- 기존 iptables보다 훨씬 빠름

**서비스 메시 통합**

- Envoy 프록시와 찰떡궁합
- 사이드카 없는 서비스 메시도 가능

**멀티 클러스터**

- 클러스터 여러 개 묶어서 관리 가능

## 둘이 만나면? 시너지 효과 💥

> _추천 이미지: [Cilium + eBPF 아키텍처](https://isovalent.com/static/ebpf-cilium-dataplane-95b1dc3e9c5b8426ee9245d4a17b0f76.png)_

Cilium과 eBPF는 완벽한 조합임:

- **데이터 플레인**: eBPF가 실제 패킷 처리, 로드밸런싱, 보안 담당
- **컨트롤 플레인**: Cilium이 정책 관리, API 제공 담당

### 실제로 어떻게 쓰는데? 💻

**네트워크 정책 예시**

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
```

**HTTP 레벨 보안 정책**

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-access-control
spec:
  endpointSelector:
    matchLabels:
      app: api-server
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: frontend
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/v1/users"
```

## 앞으로 어떻게 될까? 🔮

eBPF와 Cilium은 계속 진화 중:

**기술 발전**

- WebAssembly와 결합해서 더 안전하고 이식성 좋게
- AI/ML 워크로드 최적화
- 엣지 환경용 경량화

**생태계 확장**

- AWS, GCP, Azure 같은 클라우드에서 네이티브 지원
- 다양한 오케스트레이션 플랫폼과 연동
- 모니터링, 보안 도구들과 통합

## 마무리하며 🎬

eBPF는 리눅스 커널의 게임 체인저고, Cilium은 그 잠재력을 현실로 만든 실용적인 솔루션임. 클라우드 네이티브 환경에서 고성능, 보안, 관측성을 모두 잡고 싶다면 이 조합이 답임.

마이크로서비스와 컨테이너가 계속 늘어나는 상황에서, eBPF + Cilium의 가치는 앞으로 더욱 커질 듯.

---

**참고 링크들** 📚

- [eBPF 공식 사이트](https://ebpf.io/)
- [Cilium 공식 문서](https://docs.cilium.io/)
- [eBPF 튜토리얼](https://github.com/xdp-project/xdp-tutorial)
- [Cilium 시작하기 가이드](https://docs.cilium.io/en/stable/gettingstarted/)

**참고 자료들** 📚
![[Learning eBPF_20250718.pdf]]


- **Cilium** 관련 **추천** 정보 : **Cilium** 공식 문서 - [Link](https://docs.cilium.io/en/stable/) _← 돌고 돌아 결국 공식문서가 좋음!_
    - [Cilium] **Labs** - [Link](https://cilium.io/labs/) : 온라인 실습 Lab을 무료로 사용 가능, 추천!
        - [Isovalent] Labs - [Link](https://isovalent.com/labs/) : ‘오픈소스, 상용’ 온라인 실습 Lab을 무료로 사용 가능
    - [Cilium] **Blog** - [Link](https://cilium.io/blog/) : Cilium 기능 동작을 도식화나 표 등으로 잘 설명해둠, 추천!
        - [Isovalent] **Blog** - [Link](https://isovalent.com/blog/) : 오픈소스 링크와 동일한 주제도 있지만 다른 주제도 있으니, 같이 볼 것
    - [Cilium] Github - [Link](https://github.com/cilium/cilium) , [Tetragon](https://github.com/cilium/tetragon) , [pwru](https://github.com/cilium/pwru)
    - [Cilium] bi-weekly eCHO News - [Link](https://cilium.io/newsletter/)
    - [Youtube] **eBPF & Cilium Community** - [Link](https://www.youtube.com/@eBPFCilium) : Cilium 기능을 상세히 설명하는 영상 다수, 추천!
    - [Youtube] **eBPF: Unlocking the Kernel [OFFICIAL DOCUMENTARY]** - [Link](https://www.youtube.com/watch?v=Wb_vD3XZYOA) , [Blog](https://isovalent.com/blog/post/ebpf-documentary-creation-story/)
        - 2011년 SDN , PLUMgrid (BPF 확장 필요 → eBPF)→ RedHat 커널 패치 프로그래밍 적용
        - 커널이해하는 전문팀이 필요하여 하이버스케일 회사에서만 초기에는 사용 , 예시) Facebook L4
        - 최종 사용자에게도 eBPF 사용 경험 전달을 위해서 Cilium CNI 개발하고 이를 위해 Isovalent 회사를 창립
        - eBPF 가 Linux 이외에 Windows 에서도 동작할 수 있게 개발 되면서 → 다른 OS 등 더 폭넓게 사용을 위해 eBPF (중립) 재단 설립
    - [Youtube] CNCF **CiliumCon** 검색 - [Link](https://www.youtube.com/results?search_query=ciliumcon)
    - [eBPF] Home - [Link](https://ebpf.io/) , Blog - [Link](https://ebpf.io/blog/) , Labs - [Link](https://ebpf.io/labs/) , App - [Link](https://ebpf.io/applications/) , Infra - [Link](https://ebpf.io/infrastructure/) , Docs - [Link](https://docs.ebpf.io/)

* Isovalent Blog
* `2025`
	- **What’s New in Networking for Kubernetes in the Isovalent Platform 1.17 - [Link](https://isovalent.com/blog/post/isovalent-networking-kubernetes-1-17/)**
	- **Isovalent and Cisco ACI: Better Together - [Link](https://isovalent.com/blog/post/isovalent-cisco-aci-better-together/)**
	- **Cloud Annotations for Gateway API and Ingress with Cilium - [Link](https://isovalent.com/blog/post/cloud-annotations-for-gateway-api-ingress-with-cilium/)**
	- Remove the Chains of Kube-Proxy: Going Kube-Proxy free with Cilium - [Link](https://isovalent.com/blog/post/remove-kube-proxy-with-cilium/)
	- **Fast-Tracking Your Migration From Ingress to Gateway API - [Link](https://isovalent.com/blog/post/migrate-from-kubernetes-ingress-to-gateway-api/)**
- 2024
	- Cilium 1.16 – High-Performance Networking With Netkit, Gateway API Gamma Support, BGPV2 and More! - [Link](https://isovalent.com/blog/post/cilium-1-16/)
	- Cilium netkit: The Final Frontier in Container Networking Performance - [Link](https://isovalent.com/blog/post/cilium-netkit-a-new-container-networking-paradigm-for-the-ai-era/)
	- Cilium 1.15 – Gateway API 1.0 Support, Cluster Mesh Scale Increase, Security Optimizations and more! - [Link](https://isovalent.com/blog/post/cilium-1-15/?utm_source=website-cilium&utm_medium=referral&utm_campaign=cilium-blog)
