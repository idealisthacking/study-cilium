•	FRR는 오픈소스 라우팅 데몬 세트로, Kubernetes Node 혹은 별도 게이트웨이에서 BGP 프로토콜 작동을 담당합니다.
	•	Cilium, MetalLB, Calico 등은 내부적으로 FRR 혹은 goBGP, bird 등과 연동해 PodCIDR/Service IP를 BGP로 광고합니다.
	•	네트워크 장비와 BGP Peer를 맺어 각 Node별 CIDR이 사내 라우터/스위치에 실시간 반영.
	•	실무에서는 FRR 설정파일을 자동화 스크립트, Helm, ConfigMap, CRD 등으로 쉽게 관리합니다.