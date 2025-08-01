- 레이어 7 트래픽 모니터링에는 **사용자 이름, 비밀번호, 쿼리 매개변수, API 키** 등 잠재적으로 민감한 정보를 처리하기 위한 보안 고려 사항이 포함됩니다.
- *By default, Hubble does not redact potentially sensitive information present in Layer 7 Hubble Flows. ⇒ **허블은 민감정보 직접 편집 X***
- 간단 실습 따라해보기

```bash
#
hubble observe -f -t l7
Jul 20 09:12:57.239: default/curl-pod:39818 (ID:20680) -> default/webpod-697b545f57-mt7vg:80 (ID:35772) http-request FORWARDED (HTTP/1.1 GET http://webpod/?user_id=1234)
Jul 20 09:12:57.249: default/curl-pod:39818 (ID:20680) <- default/webpod-697b545f57-mt7vg:80 (ID:35772) http-response FORWARDED (HTTP/1.1 200 10ms (GET http://webpod/?user_id=1234))

#
kubectl exec -it curl-pod -- sh -c 'curl -s webpod/?user_id=1234'
kubectl exec -it curl-pod -- sh -c 'curl -s webpod/?user_id=1234'


# 민감정보 미출력 설정
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
  --set extraArgs="{--hubble-redact-enabled,--hubble-redact-http-urlquery}"

#
kubectl exec -it curl-pod -- sh -c 'curl -s webpod/?user_id=1234'

#
hubble observe -f -t l7
Jul 20 09:15:56.618: default/curl-pod:49882 (ID:20680) -> default/webpod-697b545f57-xt9x5:80 (ID:35772) http-request FORWARDED (HTTP/1.1 GET http://webpod/)
Jul 20 09:15:56.627: default/curl-pod:49882 (ID:20680) <- default/webpod-697b545f57-xt9x5:80 (ID:35772) http-response FORWARDED (HTTP/1.1 200 10ms (GET http://webpod/)
```
![[스크린샷 2025-07-27 오후 12.52.11.png]]

- 보안을 강화하기 위해 Cilium은 허블이 레이어 7 흐름에 존재하는 민감한 정보를 **처리(제거나 마스킹)**할 수 있도록 `--hubble-redact-enabled` 옵션을 제공합니다.
- 보다 구체적으로, 지원되는 레이어 7 프로토콜에 대해 다음과 같은 기능을 제공합니다:
    - For **HTTP**: **redacting URL query** (GET) parameters (`--hubble-redact-http-urlquery`) _⇒ URL의 `?query=value` 부분 제거_

- For **HTTP**: **redacting URL user info** (for example, **password** used in basic auth) (`-hubble-redact-http-userinfo`)
    
    _⇒ URL의 `user:pass@` 부분 제거_
    
- For **Kafka**: **redacting API key** (`-hubble-redact-kafka-apikey`)
    
- For **HTTP headers**: redacting all headers except those defined in the `-hubble-redact-http-headers-allow` list or redacting only the headers defined in the `-hubble-redact-http-headers-deny` list
