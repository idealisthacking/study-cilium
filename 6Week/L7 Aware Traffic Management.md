# [https://isovalent.com/labs/cilium-envoy-l7-proxy/](https://isovalent.com/labs/cilium-envoy-l7-proxy/) 온라인 Lab 관련 설명 참고
![[Pasted image 20250817225810.png]]
![[Pasted image 20250817225823.png]]
![[Pasted image 20250817225835.png]]
```bash
# The trace:to-proxy filter will show all flows that go through a proxy. In general, this could be either Envoy or the Cilium DNS proxy. 
# However, since we have not yet deployed a DNS Network Policy, you will only see flows related to Envoy at the moment.
hubble observe --type trace:to-proxy

# Let's extract all the flow information based on the protocol (e.g. HTTP) and source pod (e.g. the Tie Fighter), then export the result with the JSON output option, and finally filter with jq to only see the .flow.l7 field. This will show us the specific details parsed from the L7 traffic, such as the method and headers:
# Observe the details of the flow, in particular the envoy-specific headers added to the request:
## X-Envoy-Internal
## X-Request-Id
hubble observe --protocol http --from-pod default/tiefighter -o jsonpb | \
  head -n 1 | jq '.flow.l7'
{
  "type": "REQUEST",
  "http": {
    "method": "POST",
    "url": "http://deathstar.default.svc.cluster.local/v1/request-landing",
    "protocol": "HTTP/1.1",
    "headers": [
      {
        "key": ":scheme",
        "value": "http"
      },
      {
        "key": "Accept",
        "value": "*/*"
      },
      {
        "key": "User-Agent",
        "value": "curl/7.88.1"
      },
      {
        "key": "X-Envoy-Internal",
        "value": "true"
      },
      {
        "key": "X-Request-Id",
        "value": "241e4f8f-f668-49fb-9b2c-50dc8ea92b77"
      }
    ]
  }
}

# Let's use the X-Request-Id to match a request and its response.
# First, we'll need to make sure egress traffic from the Tie Fighter is captured by Envoy, so we'll need a L7 CNP for that.
# If we apply an egress CNP though, this will disrupt DNS requests, which are also egress traffic, so we need to add a DNS policy as well:
cat policies/tiefighter.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: tiefighter
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      org: empire
      class: tiefighter
  egress:
    - toEndpoints:
        - matchLabels:
            class: deathstar
            org: empire
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:
            - method: POST
              path: /v1/request-landing
              
kubectl apply -f policies/dns.yaml -f policies/tiefighter.yaml


# All these flows are ingress flows, as you can see by filtering for HTTP flows in the egress direction, which should return nothing:
# You can now see egress requests from the Tie Fighter being forwarded to the Death Star, as well as the responses from the Death Star:
# When using Kubernetes Network Policies, responses are automatically allowed and do not require an explicit rule.
# This is why egress traffic corresponding to the response from the Death Star to the Tie Fighter is allowed, even though there is not egress policy for it.
hubble observe --protocol http --traffic-direction egress
Aug 17 08:55:57.116: default/tiefighter:37494 (ID:8125) -> default/deathstar-67c5c5c88-8dx2x:80 (ID:65278) http-request FORWARDED (HTTP/1.1 POST http://deathstar.default.svc.cluster.local/v1/request-landing)
Aug 17 08:55:57.118: default/tiefighter:37494 (ID:8125) <- default/deathstar-67c5c5c88-8dx2x:80 (ID:65278) http-response FORWARDED (HTTP/1.1 200 1ms (POST http://deathstar.default.svc.cluster.local/v1/request-landing))

# Now, let's match request IDs! Run the following command to record some Hubble HTTP flows and save them to a file:
hubble observe --namespace default --protocol http -o jsonpb > flows.json

# Find the first EGRESS flow in the file and get its ID:
REQUEST_ID=$(cat flows.json | jq -r '.flow | select(.source.labels[0]=="k8s:app.kubernetes.io/name=tiefighter" and .traffic_direction=="EGRESS") .l7.http.headers[] | select(.key=="X-Request-Id") .value' | head -n1)
echo $REQUEST_ID
37c5e90f-6fcf-4111-a253-d0b0cb666832

# Then find all flows with this request ID in the file and display their source identities:
## an egress flow from the tiefighter to the deathstar, corresponding to the original request
## the ingress flow for the same request, being forwarded from the proxy to the Death Star
## another egress flow for the response, from the deathstar to the tiefighter
## the corresponding ingress flow from the deathstar pod to the tiefighter

cat flows.json | \
  jq 'select(.flow.l7.http.headers[] | .value == "'$REQUEST_ID'") .flow | {src_label: .source.labels[0], dst_label: .destination.labels[0], traffic_direction, type: .l7.type, time}'
{
  "src_label": "k8s:app.kubernetes.io/name=tiefighter",
  "dst_label": "k8s:app.kubernetes.io/name=deathstar",
  "traffic_direction": "EGRESS",
  "type": "REQUEST",
  "time": "2025-08-17T08:57:31.868753172Z"
}
{
  "src_label": "k8s:app.kubernetes.io/name=tiefighter",
  "dst_label": "k8s:app.kubernetes.io/name=deathstar",
  "traffic_direction": "INGRESS",
  "type": "REQUEST",
  "time": "2025-08-17T08:57:31.869283236Z"
}
{
  "src_label": "k8s:app.kubernetes.io/name=deathstar",
  "dst_label": "k8s:app.kubernetes.io/name=tiefighter",
  "traffic_direction": "INGRESS",
  "type": "RESPONSE",
  "time": "2025-08-17T08:57:31.869728002Z"
}
{
  "src_label": "k8s:app.kubernetes.io/name=deathstar",
  "dst_label": "k8s:app.kubernetes.io/name=tiefighter",
  "traffic_direction": "EGRESS",
  "type": "RESPONSE",
  "time": "2025-08-17T08:57:31.869908523Z"
}
```
![[Pasted image 20250817225857.png]]

## L7 Metric : Using Envoy, Hubble collections L7 metrics which can easily be exported via Prometheus.
![[Pasted image 20250817225935.png]]
![[Pasted image 20250817225950.png]]
![[Pasted image 20250817230003.png]]
![[Pasted image 20250817230021.png]]

![[Pasted image 20250817230032.png]]

# L7-Aware Traffic Management 활성화 : CiliumEnvoyConfig , CiliumClusterwideEnvoyConfig - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/l7-traffic-management/)

- 주의사항 : _**알아서 잘 쓰시라!**_
    - CiliumEnvoyConfig 리소스는 최소한의 검증만 수행되며 정의된 충돌 해결 동작이 없습니다. 즉, EnvoyConfig의 동일한 구성 부분을 수정하는 CEC를 여러 개 만들면 결과를 예측할 수 없을 수 있습니다.
    - 이 최소한의 검증 외에도, CiliumEnvoyConfig는 사용자에게 구성의 정확성에 대한 피드백을 최소한으로 제공합니다. 따라서 CEC가 바람직하지 않은 결과를 초래할 경우, 문제 해결을 위해서는 문제의 CiliumEnvoyConfig를 확인할 수 있는 대신 EnvoyConfig를 검토해야 합니다.
    - CiliumEnvoyConfig는 Cilium의 Ingress 및 Gateway API 지원을 통해 노드별 Envoy 프록시를 통해 트래픽을 유도하는 데 사용됩니다. 자동 생성된 구성과 충돌하거나 수정하는 CEC를 만들면 결과를 예측할 수 없을 수 있습니다. 이러한 사용 사례에서는 CEC를 사용하는 데 매우 주의하세요. 위의 위험은 Cilium에서 생성된 모든 구성이 가능한 한 의미적으로 유효한지 확인함으로써 관리됩니다.
    - CiliumEnvoyConfig 리소스를 직접 생성하는 경우(즉, Cilium Ingress 또는 Gateway API 컨트롤러를 통해 생성하지 않고), CEC가 E/W 트래픽을 관리하려는 경우 레이블 [cilium.io/use-original-source-address](http://cilium.io/use-original-source-address) : "false"를 설정합니다. 그렇지 않으면, EnvoyConfig는 업스트림 연결 풀의 소켓을 원래 소스 주소/포트에 바인딩합니다. 이로 인해 포드가 동일한 파이프라인 HTTP/1.1 또는 HTTP/2 연결을 통해 여러 요청을 보낼 때 5-tup 충돌이 발생할 수 있습니다. (Cilium 에이전트는 부모 Ref가 Cilium Ingress 또는 Gateway API 컨트롤러를 가리키는 모든 CEC가 [cilium.io/use-original-source-address](http://cilium.io/use-original-source-address) 을 "false"로 설정한 것으로 가정하지만, 다른 모든 CEC는 이 레이블을 "true"로 설정한 것으로 가정합니다.)
    - As of now only the **Envoy API v3 is supported**. - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/l7-traffic-management/#supported-envoy-api-versions)

## 설정
```bash
#
helm upgrade cilium cilium/cilium --version 1.18.1 --namespace kube-system --reuse-values \
--set ingressController.enabled=true --set gatewayAPI.enabled=false \
--set envoyConfig.enabled=true  --set loadBalancer.l7.backend=envoy

kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart ds/cilium
kubectl -n kube-system rollout restart ds/cilium-envoy

#
cilium config view |grep -i envoy
cilium status --wait
```

- **Supported Envoy Extension Resource Types**
    - Envoy extensions are resource types that may or may not be built in to an Envoy build. The standard types referred to in Envoy documentation, such as `type.googleapis.com/envoy.config.listener.v3.Listener`, and `type.googleapis.com/envoy.config.route.v3.RouteConfiguration`, are always available.
    - Cilium nodes deploy an Envoy image to support Cilium HTTP policy enforcement and observability. This build of Envoy has been optimized for the needs of the Cilium Agent and does not contain many of the Envoy extensions available in the Envoy code base.
    - To see which Envoy extensions are available, please have a look at the [Envoy extensions configuration file](https://github.com/cilium/proxy/blob/main/envoy_build_config/extensions_build_config.bzl). Only the extensions that have not been commented out with `#` are built in to the Cilium Envoy image. We will evolve the list of built-in extensions based on user feedback.

# L7 Load Balancing and URL re-writing - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/envoy-traffic-management/)

목표 : 두 백엔드 서비스(echo-service-1/2) 간의 요청 부하를 분산 및 URL Re-write Envoy 리스너를 설정
```bash
# 샘플 애플리케이션 배포
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.18.1/examples/kubernetes/servicemesh/envoy/test-application.yaml

# 확인
## 두 개의 클라이언트 배포 client , client2
## 두 가지 서비스, echo-service-1, echo-service-2
kubectl get -f https://raw.githubusercontent.com/cilium/cilium/1.18.1/examples/kubernetes/servicemesh/envoy/test-application.yaml
NAME                          DATA   AGE
configmap/coredns-configmap   1      10m

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/client           1/1     1            1           10m
deployment.apps/client2          1/1     1            1           10m
deployment.apps/echo-service-1   1/1     1            1           10m
deployment.apps/echo-service-2   1/1     1            1           10m

NAME                     TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/echo-service-1   NodePort   10.96.237.117   <none>        8080:31169/TCP   10m
service/echo-service-2   NodePort   10.96.175.3     <none>        8080:32040/TCP   10m

# client2 에 대한 Pod ID로 환경 변수 지정
export CLIENT2=$(kubectl get pods -l name=client2 -o jsonpath='{.items[0].metadata.name}')
echo $CLIENT2


# Start Observing Traffic with Hubble
cilium hubble port-forward&
hubble observe --from-pod $CLIENT2 -f


# You should be able to get a response from both of the backend services individually from client2:
kubectl exec -it $CLIENT2 -- curl -v echo-service-1:8080/
kubectl exec -it $CLIENT2 -- curl -v echo-service-2:8080/


# Verify that you get a 404 error response if you curl to the non-existent URL /foo on these services:
kubectl exec -it $CLIENT2 -- curl -v echo-service-1:8080/foo
kubectl exec -it $CLIENT2 -- curl -v echo-service-2:8080/foo 
< HTTP/1.1 404 Not Found
```

## Add Layer 7 Policy
```bash
# Adding a Layer 7 policy introduces the Envoy proxy into the path for this traffic
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.18.1/examples/kubernetes/servicemesh/envoy/client-egress-l7-http.yaml
# client2 is allowed to contact one.one.one.one/ on port 80 and the echo Pod
# on port 8080. HTTP introspection is enabled for client2.
# The toFQDNs section relies on DNS introspection being performed by
# the client-egress-only-dns policy.
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: client-egress-l7-http
spec:
  description: "Allow GET one.one.one.one:80/ and GET <echo>:8080/ from client2"
  endpointSelector:
    matchLabels:
      other: client
  egress:
    # Allow GET / requests towards echo pods.
    - toEndpoints:
        - matchLabels:
            k8s:kind: echo
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: "GET"
                path: "/"
    # Allow GET / requests, only towards one.one.one.one.
    - toFQDNs:
        - matchName: "one.one.one.one"
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:
              - method: "GET"
                path: "/"
                
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.18.1/examples/kubernetes/servicemesh/envoy/client-egress-only-dns.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: client-egress-only-dns
spec:
  endpointSelector:
    matchLabels:
      kind: client
  egress:
    - toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
      toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s:k8s-app: kube-dns
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s:k8s-app: coredns

# Make a request to a backend service (either will do):
kubectl exec -it $CLIENT2 -- curl -v echo-service-1:8080/
kubectl exec -it $CLIENT2 -- curl -v echo-service-2:8080/foo
< HTTP/1.1 403 Forbidden

# 레이어 7 정책을 추가하면 레이어 7 가시성이 가능합니다. 이제 허블 출력에 프록시로의 흐름이 포함되어 있으며, 레벨 7에서의 HTTP 프로토콜 정보도 표시됩니다
# 예: HTTP/1.1 GET http://echo-service-1:8080/
```

## Test Layer 7 Policy Enforcement
```bash
# The policy only permits GET requests to the / path, so you will see requests to any other URL being dropped. For example, try:
kubectl exec -it $CLIENT2 -- curl -v echo-service-1:8080/foo
## The Hubble output will show the HTTP request being dropped, like this:
Aug 17 09:43:55.918: default/client2-c97ddf6cf-8swbm:44564 (ID:16631) -> default/echo-service-1-867d69c679-w4hvn:8080 (ID:2665) http-request DROPPED (HTTP/1.1 GET http://echo-service-1:8080/foo)

```

## Add Envoy load-balancing and URL re-writing
```bash
# Envoy 로드 밸런싱 및 URL 재작성 추가 : 두 백엔드 echo-서비스 간에 50/50의 부하 분산 , 경로 /foo를 다시 작성합니다/
# Apply the envoy-traffic-management-test.yaml file, which defines a CiliumClusterwideEnvoyConfig
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.18.1/examples/kubernetes/servicemesh/envoy/envoy-traffic-management-test.yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideEnvoyConfig
metadata:
  name: envoy-lb-listener
spec:
  services:
    - name: echo-service-1
      namespace: default
    - name: echo-service-2
      namespace: default
  resources:
    - "@type": type.googleapis.com/envoy.config.listener.v3.Listener
      name: envoy-lb-listener
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: envoy-lb-listener
                rds:
                  route_config_name: lb_route
                use_remote_address: true
                skip_xff_append: true
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
    - "@type": type.googleapis.com/envoy.config.route.v3.RouteConfiguration
      name: lb_route
      virtual_hosts:
        - name: "lb_route"
          domains: [ "*" ]
          routes:
            - match:
                prefix: "/"
              route:
                weighted_clusters:
                  clusters:
                    - name: "default/echo-service-1"
                      weight: 50
                    - name: "default/echo-service-2"
                      weight: 50
                retry_policy:
                  retry_on: 5xx
                  num_retries: 3
                  per_try_timeout: 1s
                regex_rewrite:
                  pattern:
                    google_re2: { }
                    regex: "^/foo.*$"
                  substitution: "/"
    - "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
      name: "default/echo-service-1"
      connect_timeout: 5s
      lb_policy: ROUND_ROBIN
      type: EDS
      outlier_detection:
        split_external_local_origin_errors: true
        consecutive_local_origin_failure: 2
    - "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
      name: "default/echo-service-2"
      connect_timeout: 3s
      lb_policy: ROUND_ROBIN
      type: EDS
      outlier_detection:
        split_external_local_origin_errors: true
        consecutive_local_origin_failure: 2

# CiliumClusterwideEnvoyConfig 약자 ccec
kubectl get ccec -o yaml | yq
kubectl get ccec 
NAME                AGE
envoy-lb-listener   10s

# 50:50 LB 분산 호출됨
kubectl exec -it $CLIENT2 -- curl -v echo-service-1:8080

# A request to /foo should now succeed, because of the path re-writing:
kubectl exec -it $CLIENT2 -- curl -v echo-service-1:8080/foo


# 하지만 네트워크 정책은 여전히 /로 다시 작성되지 않은 경로에 대한 요청을 방지합니다. 
# 예를 들어, 이 요청은 패킷이 삭제되고 403 금지 응답 코드가 생성
kubectl exec -it $CLIENT2 -- curl -v echo-service-1:8080/bar
< HTTP/1.1 403 Forbidden
## Output from hubble observe
Aug 17 09:47:32.745: default/client2-c97ddf6cf-8swbm:54094 (ID:16631) -> default/echo-service-1-867d69c679-w4hvn:8080 (ID:2665) http-request DROPPED (HTTP/1.1 GET http://echo-service-1:8080/bar)

# 로그 확인
kubectl -n kube-system logs daemonsets/cilium-envoy

```

# 도전과제4-1 L7 Path Translation - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/envoy-custom-listener/)

^5a81d3

# 도전과제4-2 L7 Circuit Breaking - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/envoy-circuit-breaker/)

^bda7f9

# 도전과제4-3 Proxy Load Balancing for Kubernetes Services (beta) - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/envoy-load-balancing/)

^4642f2

# 도전과제4-4 L7 Traffic Shifting - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/envoy-traffic-shifting/)

^09d7f3

