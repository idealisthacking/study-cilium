# 적용하는 exporter 필터/튜닝 항목
* allowList : 정말 필요한 verdict(예: DROPPED, ERROR, 또는 특정 HTTP code 등)만 포함
	* 예시: –set hubble.export.static.allowList=’{“verdict”:“DROPPED”,“ERROR”}’ 필요에 따라 “FORWARDED” 등 추가 가능
* denyList : 불필요하거나 잡음을 많이 유발하는 소스/타입 식별 후 배제
	* 예시: –set hubble.export.static.denyList=’{“source_pod”:“kube-system/”}’ system namespace, 특정 서비스 제외 {“destination_namespace”:“kube-system”} {“source_ip”:“10.0.0.1”} {“event_type”:“trace”} 등
* rateLimit : 초당 최대 메시지 수 설정(예: –set hubble.export.rateLimit=100)
* fieldMask : 불필요한 필드 제외, 피크 트래픽/디스크 사용량 줄이기
* excludeFlowTypes : L7, DNS 등 특정 타입 트래픽 제외 

```bash
helm upgrade --install cilium cilium/cilium \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.export.enabled=true \
  --set hubble.export.static.allowList[0]='{"verdict":["DROPPED","ERROR"]}' \
  --set hubble.export.static.allowList[1]='{"destination_namespace":["prod"]}' \
  --set hubble.export.static.denyList[0]='{"source_namespace":["kube-system","kube-public"]}' \
  --set hubble.export.static.denyList[1]='{"event_type":["trace"]}'
```

# Logging Tip
* DROPPED/ERROR만 저장
* prod Namespace만 export (dev/stg는 제외)
* kube-system, kube-public 등은 완전 배제
* trace, debug 등 대량 noisy event 배제
* 로그 볼륨 초과 시 fieldMask나 excludeFlowTypes 활용
* 허용 리스트가 너무 넓으면 disk/tps 폭주 주의 (inverse logic: 필요한 것만 allow)
