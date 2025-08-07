# 파드 생성 : K8S 클러스터 내부에서만 접속
![[Pasted image 20250803205803.png]]

# 서비스(Cluster Type) 연결 : K8S 클러스터 내부에서만 접속
- 동일한 애플리케이션의 다수의 파드의 접속을 용이하게 하기 위한 서비스에 접속
- 고정 접속(호출) 방법을 제공 : 흔히 말하는 ‘고정 VirtualIP’ 와 ‘Domain주소’ 생성
![[Pasted image 20250803205826.png]]

# 서비스(NodePort Type) 연결 : 외부 클라이언트가 서비스를 통해서 클러스터 내부의 파드로 접속
- 서비스(NodePort Type)의 일부 단점을 보완한 서비스(LoadBalancer Type) 도 있습니다

![[Pasted image 20250803205908.png]]


# Service 종류
## ClusterIP 타입
![[Pasted image 20250803210043.png]]

## NodePort 타입
![[Pasted image 20250803210103.png]]

## LoadBalancer 타입
![[Pasted image 20250803210124.png]]

#  eBPF 모드 + XDP
   - 기존 netfilter/iptables 기반 통신
![[Pasted image 20250803210316.png]]

![[Pasted image 20250803210331.png]]

