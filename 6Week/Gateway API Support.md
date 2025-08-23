# Gateway API의 주요 기능 : 현재 최신 v1.3.0, N-S(Ingress) 와 E-S(Service Mesh, GAMMA Initiative) - [Home](https://gateway-api.sigs.k8s.io/)

1. **개선된 리소스 모델**
    : API는 GatewayClass, Gateway 및 Route(HTTPRoute, TCPRoute 등)와 같은 새로운 사용자 정의 리소스를 도입하여 라우팅 규칙을 정의하는 보다 세부적이고 표현력 있는 방법을 제공합니다.
2. **프로토콜 독립적**
    : 주로 HTTP용으로 설계된 Ingress와 달리 Gateway API는 TCP, UDP, TLS를 포함한 여러 프로토콜을 지원합니다.
3. **강화된 보안**
    : TLS 구성 및 보다 세부적인 액세스 제어에 대한 기본 제공 지원.
4. **교차 네임스페이스 지원**
    : 서로 다른 네임스페이스의 서비스로 트래픽을 라우팅하여 보다 유연한 아키텍처를 구축할 수 있는 기능을 제공합니다.
5. **확장성**
    : API는 사용자 정의 리소스 및 정책으로 쉽게 확장할 수 있도록 설계되었습니다.
6. **역할 지향**
    : 클러스터 운영자, 애플리케이션 개발자, 보안 팀 간의 우려를 명확하게 분리합니다.
    이것은 제 겸손한 의견으로는 Gateway API의 가장 흥미로운 기능 중 하나일 것입니다.

# Gateway API 소개

: 기존의 Ingress 에 좀 더 기능을 추가, 역할 분리(role-oriented) - [Docs](https://kubernetes.io/docs/concepts/services-networking/gateway/) , [Youtube](https://www.youtube.com/watch?v=kbTVzAhtHKs)
- 서비스 메시(istio)에서 제공하는 Rich 한 기능 중 일부 기능들과 혹은 운영 관리에 필요한 기능들을 추가
- 추가 기능 : 헤더 기반 라우팅, 헤더 변조, 트래픽 미러링(쉽게 트래픽 복제), 역할 기반
![[Pasted image 20250817213134.png]]

- **Gateway API** is a family of API kinds that provide dynamic infrastructure provisioning and advanced **traffic routing.**
- Make network services available by using an extensible, **role-oriented**, protocol-aware configuration mechanism.
- [Gateway API](https://gateway-api.sigs.k8s.io/) is an [add-on](https://kubernetes.io/docs/concepts/cluster-administration/addons/) containing API [kinds](https://gateway-api.sigs.k8s.io/references/spec/) that provide dynamic infrastructure provisioning and advanced traffic routing.

# 구성 요소 (Resource)

GatewayClass, Gateway, HTTPRoute, TCPRoute, Service
![[Pasted image 20250817213210.png]]
- **GatewayClass:** Defines a set of gateways with common configuration and managed by a controller that implements the class.
- **Gateway:** Defines an instance of traffic handling infrastructure, such as cloud load balancer.
- **HTTPRoute:** Defines HTTP-specific rules for mapping traffic from a Gateway listener to a representation of backend network endpoints. These endpoints are often represented as a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

![[Pasted image 20250817213233.png]]

![[Pasted image 20250817213401.png]]'
![[Pasted image 20250817213414.png]]

Kubernetes Traffic Management: Combining Gateway API with Service Mesh for North-South and East-West Use Cases - [Blog](https://medium.com/@disha.20.10/kubernetes-traffic-management-combining-gateway-api-with-service-mesh-for-north-south-and-63e39ad95dcc)

Request flow
![[Pasted image 20250817213433.png]]

Why does a role-oriented API matter? - [Blog1](https://www.anyflow.net/sw-engineer/kubernetes-gateway-api-1), [Blog2](https://www.anyflow.net/sw-engineer/kubernetes-gateway-api-2)

- 담당 업무의 역할에 따라서 동작/권한을 유연하게 제공할 수 있음
- 아래 그림 처럼 '스토어 개발자'는 Store 네임스페이스내에서 해당 store PATH 라우팅 관련 정책을 스스로 관리 할 수 있음
![[Pasted image 20250817213511.png]]
- **Infrastructure Provider:** Manages infrastructure that allows multiple isolated clusters to serve multiple tenants, e.g. a cloud provider.
- **Cluster Operator:** Manages clusters and is typically concerned with policies, network access, application permissions, etc.
- **Application Developer:** Manages an application running in a cluster and is typically concerned with application-level configuration and [Service](https://kubernetes.io/docs/concepts/services-networking/service/) composition.

# Cilium Gateway API Support 소개 및 설정 - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/)
Ingress 제약 : 고급 라우팅 지원 부족(URL rewriting 등), TCP/UDP 등 프로토콜 지원 부족, 세부 권한 분리 부족

- **Cilium** supports **Gateway API v1.2.0** for below resources, all the Core conformance tests are passed. → v1.3은 아직 일부 항목 만족 X [https://gateway-api.sigs.k8s.io/implementations/v1.3/](https://gateway-api.sigs.k8s.io/implementations/v1.3/)
    - [GatewayClass](https://gateway-api.sigs.k8s.io/api-types/gatewayclass/)
    - [Gateway](https://gateway-api.sigs.k8s.io/api-types/gateway/)
    - [HTTPRoute](https://gateway-api.sigs.k8s.io/api-types/httproute/)
    - [GRPCRoute](https://gateway-api.sigs.k8s.io/api-types/grpcroute/)
    - [TLSRoute (experimental)](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1alpha2.TLSRoute)
    - [ReferenceGrant](https://gateway-api.sigs.k8s.io/api-types/referencegrant/)
        - Additionally, Cilium provides `CiliumGatewayClassConfig` CRD, which can be referenced in [GatewayClass.parametersRef](https://gateway-api.sigs.k8s.io/api-types/gatewayclass/#gatewayclass-parameters).

- 사전준비
    - Cilium must be configured with NodePort enabled, using `nodePort.enabled=true` or by enabling the kube-proxy replacement with `kubeProxyReplacement=true`. For more information, see [kube-proxy replacement](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#kubeproxy-free).
    - Cilium must be configured with the L7 proxy enabled using `l7Proxy=true` (enabled by default).
    - The below CRDs from Gateway API v1.2.0 `must` be pre-installed. Please refer to this [docs](https://gateway-api.sigs.k8s.io/guides/?h=crds#getting-started-with-gateway-api) for installation steps. Alternatively, the below snippet could be used. _**→ CRD 설치 필수!**_

```bash
# CRD 설치
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_gateways.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml

# 확인
kubectl get crd | grep gateway.networking.k8s.io
gatewayclasses.gateway.networking.k8s.io     2025-08-17T04:27:28Z
gateways.gateway.networking.k8s.io           2025-08-17T04:27:28Z
grpcroutes.gateway.networking.k8s.io         2025-08-17T04:27:30Z
httproutes.gateway.networking.k8s.io         2025-08-17T04:27:29Z
referencegrants.gateway.networking.k8s.io    2025-08-17T04:27:29Z
tlsroutes.gateway.networking.k8s.io          2025-08-17T04:27:30Z
```
By default, the Gateway API controller creates a service of LoadBalancer type, so your environment will need to support this. Alternatively, since Cilium 1.16+, you can directly expose the Cilium L7 proxy on the [host network](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/#gs-gateway-host-network-mode).


# Cilium Gateway API 설정
```bash
#
helm upgrade cilium cilium/cilium --version 1.18.1 --namespace kube-system --reuse-values \
--set ingressController.enabled=false --set gatewayAPI.enabled=true

#
kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart ds/cilium

#
cilium config view | grep gateway-api
enable-gateway-api                                true
enable-gateway-api-alpn                           false
enable-gateway-api-app-protocol                   false
enable-gateway-api-proxy-protocol                 false
enable-gateway-api-secrets-sync                   true
gateway-api-hostnetwork-enabled                   false
gateway-api-hostnetwork-nodelabelselector         
gateway-api-secrets-namespace                     cilium-secrets
gateway-api-service-externaltrafficpolicy         Cluster
gateway-api-xff-num-trusted-hops                  0

# cilium-ingress 제거 확인
kubectl get svc,pod -n kube-system

# The GatewayClass is a type of Gateway that can be deployed: in other words, it is a template. This is done in a way to let infrastructure providers offer different types of Gateways. Users can then choose the Gateway they like.
# For instance, an infrastructure provider may create two GatewayClasses named internet and private to reflect Gateways that define Internet-facing vs private, internal applications.
# In our case, the Cilium Gateway API (io.cilium/gateway-controller) will be instantiated.
# This schema below represents the various components used by Gateway APIs. When using Ingress, all the functionalities were defined in one API. By deconstructing the ingress routing requirements into multiple APIs, users benefit from a more generic, flexible and role-oriented model
# The actual L7 traffic rules are defined in the HTTPRoute API.
# In the next challenge, you will deploy an application and set up GatewayAPI HTTPRoutes to route HTTP traffic into the cluster.
kubectl get GatewayClass
NAME     CONTROLLER                     ACCEPTED   AGE
cilium   io.cilium/gateway-controller   True       2m19s

kubectl get gateway -A
No resources found
```

- **Underlying mechanics: a high level overview - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/#underlying-mechanics-a-high-level-overview)**
    - A Cilium deployment has two parts that handle **Gateway API resources**: the **Cilium** **agent** and the **Cilium** **operator**.
    - The **Cilium operator** **watches** all **Gateway API resource**s and **verifies** whether the resources are valid. If resources are valid, the operator marks them as accepted. That starts the process of translation into Cilium Envoy Configuration resources.
    - The **Cilium agent** then picks up the **Cilium Envoy Configuration** resources.
    - The **Cilium agent** uses the resources to supply the configuration to the built in Envoy or the Envoy DaemonSet. Envoy handles traffic.

# Deploy the Cilium Gateway
```bash
#
cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: cilium
  listeners:
  - protocol: HTTP
    port: 80
    name: web-gw
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-app-1
spec:
  parentRefs:
  - name: my-gateway
    namespace: default
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /details
    backendRefs:
    - name: details
      port: 9080
  - matches:
    - headers:
      - type: Exact
        name: magic
        value: foo
      queryParams:
      - type: Exact
        name: great
        value: example
      path:
        type: PathPrefix
        value: /
      method: GET
    backendRefs:
    - name: productpage
      port: 9080
EOF
```

```bash
# You will see a LoadBalancer service named cilium-gateway-my-gateway which was created for the Gateway API.
kubectl get svc,ep cilium-gateway-my-gateway
NAME                                TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
service/cilium-gateway-my-gateway   LoadBalancer   10.96.240.104   192.168.10.211   80:31017/TCP   96s

NAME                                  ENDPOINTS              AGE
endpoints/cilium-gateway-my-gateway   192.192.192.192:9999   96s

# The same external IP address is also associated to the Gateway:
kubectl get gateway
NAME         CLASS    ADDRESS          PROGRAMMED   AGE
my-gateway   cilium   192.168.10.211   True         14s

## Accepted: the Gateway configuration was accepted.
## Programmed: the Gateway configuration was programmed into Envoy.
## ResolvedRefs: all referenced secrets were found and have permission for use.
kc describe gateway

#
kubectl get httproutes -A
NAMESPACE   NAME         HOSTNAMES   AGE
default     http-app-1               8m12s

# Accepted: The HTTPRoute configuration was correct and accepted.
# ResolvedRefs: The referenced services were found and are valid references.
kc describe httproutes

# Check Cilium Operator logs
kubectl logs -n kube-system deployments/cilium-operator | grep gateway

```

# Make Requests
HTTP Path matching & HTTP Header Matching
```bash
#
GATEWAY=$(kubectl get gateway my-gateway -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY

# HTTP Path matching
# Let's now check that traffic based on the URL path is proxied by the Gateway API.
# Check that you can make HTTP requests to that external address:
# Because the path starts with /details, this traffic will match the first rule and will be proxied to the details Service over port 9080.
curl --fail -s http://"$GATEWAY"/details/1 | jq
sshpass -p 'vagrant' ssh vagrant@router "curl -s --fail -v http://"$GATEWAY"/details/1"

# HTTP Header Matching
# This time, we will route traffic based on HTTP parameters like header values, method and query parameters. Run the following command:
curl -v -H 'magic: foo' http://"$GATEWAY"\?great\=example
sshpass -p 'vagrant' ssh vagrant@router "curl -s -v -H 'magic: foo' http://"$GATEWAY"\?great\=example"
```

# HTTPS Example - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/https/)

```bash
#
cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tls-gateway
spec:
  gatewayClassName: cilium
  listeners:
  - name: https-1
    protocol: HTTPS
    port: 443
    hostname: "bookinfo.cilium.rocks"
    tls:
      certificateRefs:
      - kind: Secret
        name: demo-cert
  - name: https-2
    protocol: HTTPS
    port: 443
    hostname: "webpod.cilium.rocks"
    tls:
      certificateRefs:
      - kind: Secret
        name: demo-cert
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-app-route-1
spec:
  parentRefs:
  - name: tls-gateway
  hostnames:
  - "bookinfo.cilium.rocks"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /details
    backendRefs:
    - name: details
      port: 9080
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-app-route-2
spec:
  parentRefs:
  - name: tls-gateway
  hostnames:
  - "webpod.cilium.rocks"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: webpod
      port: 80
EOF
```

```bash
#
kubectl get gateway tls-gateway
NAME          CLASS    ADDRESS          PROGRAMMED   AGE
tls-gateway   cilium   192.168.10.213   True         9m25s

kubectl get httproutes https-app-route-1 https-app-route-2
NAME                HOSTNAMES                   AGE
https-app-route-1   ["bookinfo.cilium.rocks"]   9m43s
https-app-route-2   ["webpod.cilium.rocks"]     9m43s

#
GATEWAY2=$(kubectl get gateway tls-gateway -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY2

curl -s --resolve bookinfo.cilium.rocks:443:${GATEWAY2} https://bookinfo.cilium.rocks/details/1 | jq
  
curl -s --resolve webpod.cilium.rocks:443:${GATEWAY2}   https://webpod.cilium.rocks/ -v

```

# TLS Route : Terminate vs. Passthrough
![[Pasted image 20250817213814.png]]
- **In Terminate:**
    - Client → Gateway: HTTPS
    - Gateway → Pod: HTTP
- **In Passthrough:**
    - Client → Gateway: HTTPS
    - Gateway → Pod: HTTPS

# Deploy Sample App
```bash
# Deploy the Demo app : HTTPS 웹서버
# We will be using a NGINX web server. Review the NGINX configuration.
cat <<'EOF' > nginx.conf
events {
}

http {
  log_format main '$remote_addr - $remote_user [$time_local]  $status '
  '"$request" $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$http_x_forwarded_for"';
  access_log /var/log/nginx/access.log main;
  error_log  /var/log/nginx/error.log;

  server {
    listen 443 ssl;

    root /usr/share/nginx/html;
    index index.html;

    server_name nginx.cilium.rocks;
    ssl_certificate /etc/nginx-server-certs/tls.crt;
    ssl_certificate_key /etc/nginx-server-certs/tls.key;
  }
}
EOF

# As you can see, it listens on port 443 for SSL traffic. Notice it specifies the certificate and key previously created.
# We will need to mount the files to the right path (/etc/nginx-server-certs) when we deploy the server.
# The NGINX server configuration is held in a Kubernetes ConfigMap. Let's create it.
kubectl create configmap nginx-configmap --from-file=nginx.conf=./nginx.conf

# Review the NGINX server Deployment and the Service fronting it:
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
    - port: 443
      protocol: TCP
  selector:
    run: my-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
        - name: my-nginx
          image: nginx
          ports:
            - containerPort: 443
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx
              readOnly: true
            - name: nginx-server-certs
              mountPath: /etc/nginx-server-certs
              readOnly: true
      volumes:
        - name: nginx-config
          configMap:
            name: nginx-configmap
        - name: nginx-server-certs
          secret:
            secretName: demo-cert
EOF

```

```bash
# Verify the Service and Deployment have been deployed successfully:
kubectl get deployment,svc,ep my-nginx
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-nginx   1/1     1            1           48s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/my-nginx   ClusterIP   10.96.142.252   <none>        443/TCP   48s

NAME                 ENDPOINTS         AGE
endpoints/my-nginx   172.20.1.59:443   48s
```

# Deploy the Gateway
```bash
# 
cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cilium-tls-gateway
spec:
  gatewayClassName: cilium
  listeners:
    - name: https
      hostname: "nginx.cilium.rocks"
      port: 443
      protocol: TLS
      tls:
        mode: Passthrough
      allowedRoutes:
        namespaces:
          from: All
---
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: nginx
spec:
  parentRefs:
    - name: cilium-tls-gateway
  hostnames:
    - "nginx.cilium.rocks"
  rules:
    - backendRefs:
        - name: my-nginx
          port: 443
EOF
```

```bash

# The Gateway does not actually inspect the traffic aside from using the SNI header for routing. Indeed the hostnames field defines a set of SNI names that should match against the SNI attribute of TLS ClientHello message in TLS handshake.
# Let's now deploy the Gateway and the TLSRoute to the cluste
kubectl get gateway cilium-tls-gateway
NAME                 CLASS    ADDRESS          PROGRAMMED   AGE
cilium-tls-gateway   cilium   172.18.255.202   True         8s

GATEWAY=$(kubectl get gateway cilium-tls-gateway -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY

# Let's also double check the TLSRoute has been provisioned successfully and has been attached to the Gateway.
kubectl get tlsroutes.gateway.networking.k8s.io -o json | jq '.items[0].status.parents[0]'
...
```

# Make TLS requests
```bash
# The data should be properly retrieved, using HTTPS (and thus, the TLS handshake was properly achieved).
# There are several things to note in the output.
## It should be successful (you should see at the end, a HTML output with Cilium rocks.).
## The connection was established over port 443 - you should see Connected to nginx.cilium.rocks (172.18.255.200) port 443 .
## You should see TLS handshake and TLS version negotiation. Expect the negotiations to have resulted in TLSv1.3 being used.
## Expect to see a successful certificate verification (look out for SSL certificate verify ok).
curl -v --resolve "nginx.cilium.rocks:443:$GATEWAY" "https://nginx.cilium.rocks:443"

# (추가) nginx 파드 내에서 tcpudmp 나 access log 확인해보기!
```

# Gateway API Address Support : GW EX-IP 직접 지정 - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/#gateway-api-addresses-support)
k9s → 아무 gateway Edit

```bash
spec:
  addresses:
  - type: IPAddress
    value: 192.168.10.215
  gatewayClassName: cilium
...
```

호출 테스트!
```bash
#
GATEWAY=$(kubectl get gateway cilium-tls-gateway -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY

curl -v --resolve "nginx.cilium.rocks:443:$GATEWAY" "https://nginx.cilium.rocks:443"
```

# Migrating from Ingress to Gateway : 기존 Ingress 를 → Gateway API로 마이그레이션 - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/ingress-to-gateway/ingress-to-gateway/)

- _**manual**_: manually creating Gateway API resources based on existing Ingress API resources.
- _**automated**_: creating rules using the [ingress2gateway tool](https://github.com/kubernetes-sigs/ingress2gateway). The ingress2gateway project reads Ingress resources from a Kubernetes cluster based on your current Kube Config. It outputs YAML for equivalent Gateway API resources to stdout.
    - The `ingress2gateway` tool remains experimental and is not recommended for production. - [Github](https://github.com/kubernetes-sigs/ingress2gateway)
- Example
    - HTTP Migration Example - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/ingress-to-gateway/http-migration/)
    - TLS Migration - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/ingress-to-gateway/tls-migration/)


# `도전과제2-1` ~~Ingress gRPC Example~~ : arm64 CPU 미지원 - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/grpc/)

^8c0375

# `도전과제2-2` Traffic Splitting Example : `backendRefs.**weight:90**` - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/splitting/)

^c85819

# `도전과제2-3` HTTP Header Modifier Examples : `filters.type.**RequestHeaderModifier**` - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/header/)

^731ca8

# `도전과제2-4` GatewayClass Parameters Support - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/parameterized-gatewayclass/)

^3dff43

# `도전과제2-5` GAMMA Support : 실습은 별도 없어서, 직접 실습 내용은 검색 후 해보기 - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gamma/)

^034a5c

# `도전과제2-6` Isovalent 에 Gateway API 온라인 실습 따라해보고 정리하기 - [GW_Lab1](https://isovalent.com/labs/cilium-gateway-api/) , [GW_Lab2](https://isovalent.com/labs/cilium-gateway-api-advanced/)

^9bdd42

# `도전과제2-7` Migrating from Ingress to Gateway : 기존 Ingress 를 → Gateway API로 마이그레이션 - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/ingress-to-gateway/ingress-to-gateway/)

^9de7af

- HTTP Migration Example - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/ingress-to-gateway/http-migration/)
- TLS Migration - [Docs](https://docs.cilium.io/en/stable/network/servicemesh/ingress-to-gateway/tls-migration/)

# Gateway 삭제  
```bash
kubectl delete gateway my-gateway tls-gateway cilium-tls-gateway
```

# Bookinfo 삭제
```bash 
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml
```
