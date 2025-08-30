# 실습 환경 구성 by kind k8s + Cilium CNI
```bash
# Prometheus Target connection refused bind-address 설정 : kube-controller-manager , kube-scheduler , etcd , kube-proxy
kind create cluster --name myk8s --image kindest/node:v1.33.2 --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
  - containerPort: 30001
    hostPort: 30001
  - containerPort: 30002
    hostPort: 30002
  - containerPort: 30003
    hostPort: 30003
  kubeadmConfigPatches: # Prometheus Target connection refused bind-address 설정
  - |
    kind: ClusterConfiguration
    controllerManager:
      extraArgs:
        bind-address: 0.0.0.0
    etcd:
      local:
        extraArgs:
          listen-metrics-urls: http://0.0.0.0:2381
    scheduler:
      extraArgs:
        bind-address: 0.0.0.0
  - |
    kind: KubeProxyConfiguration
    metricsBindAddress: 0.0.0.0
networking:
  disableDefaultCNI: true
  kubeProxyMode: none
  podSubnet: "10.244.0.0/16"   # cluster-cidr
kubeadmConfigPatches:
- |
  kind: ClusterConfiguration
  controllerManager:
    extraArgs:
      allocate-node-cidrs: "true"
      cluster-cidr: "10.244.0.0/16"
      node-cidr-mask-size: "22"
EOF

# node 별 PodCIDR 확인
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'


# cilium cni 설치
cilium install --version 1.18.1 --set ipam.mode=kubernetes --set ipv4NativeRoutingCIDR=172.20.0.0/16 \
--set routingMode=native --set autoDirectNodeRoutes=true --set endpointRoutes.enabled=true --set directRoutingSkipUnreachable=true \
--set kubeProxyReplacement=true --set bpf.masquerade=true \
--set endpointHealthChecking.enabled=false --set healthChecking=false \
--set hubble.enabled=true --set hubble.relay.enabled=true --set hubble.ui.enabled=true \
--set hubble.ui.service.type=NodePort --set hubble.ui.service.nodePort=30003 \
--set prometheus.enabled=true --set operator.prometheus.enabled=true --set envoy.prometheus.enabled=true --set hubble.metrics.enableOpenMetrics=true \
--set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}" \
--set debug.enabled=true  # --dry-run-helm-values

# hubble ui
open http://127.0.0.1:30003


# metrics-server
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm upgrade --install metrics-server metrics-server/metrics-server --set 'args[0]=--kubelet-insecure-tls' -n kube-system

# 확인
kubectl top node
kubectl top pod -A --sort-by='cpu'
kubectl top pod -A --sort-by='memory'

```

# Install Prometheus & Grafana - [Docs](https://docs.cilium.io/en/stable/observability/grafana/)
```bash
#
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.18.1/examples/kubernetes/addons/prometheus/monitoring-example.yaml

#
kubectl get deploy,pod,svc,ep -n cilium-monitoring
kubectl get cm -n cilium-monitoring
kc describe cm -n cilium-monitoring prometheus
kc describe cm -n cilium-monitoring grafana-config
kubectl get svc -n cilium-monitoring

# NodePort 설정
kubectl patch svc -n cilium-monitoring prometheus -p '{"spec": {"type": "NodePort", "ports": [{"port": 9090, "targetPort": 9090, "nodePort": 30001}]}}'
kubectl patch svc -n cilium-monitoring grafana -p '{"spec": {"type": "NodePort", "ports": [{"port": 3000, "targetPort": 3000, "nodePort": 30002}]}}'

# 접속 주소 확인
open "http://127.0.0.1:30001"  # prometheus
open "http://127.0.0.1:30002"  # grafana
```

# [2주차] Cilium Metrics 대시보드 & 간단 쿼리문 알아보기! : Generic, API, Cilium(BPF, kvstore, NW info, Endpoints, k8s integration)

## map ops (average node) 패널 분석
![[Pasted image 20250826150618.png]]

## 아래 쿼리문 이해해보자!
```bash
topk(5, avg(rate(cilium_bpf_map_ops_total{k8s_app="cilium", pod=~"$pod"}[5m])) by (pod, map_name, operation))
```

- **프로메테우스에서** 쿼리 해보면서 이해하기 - [Docs](https://docs.cilium.io/en/stable/observability/metrics/)
    - **cilium_bpf_map_ops_total** : Number of **eBPF map operations performed**. ⇒ 수행된 eBPF Map 작업 수
        - `mapName` is deprecated and will be removed in 1.10. Use `map_name` instead.
```bash
#
cilium_bpf_map_ops_total
cilium_bpf_map_ops_total{k8s_app="cilium"}
cilium_bpf_map_ops_total{k8s_app="cilium", pod="cilium-4hghz"}

# 최근 5분 간의 데이터로 증가율 계산
rate(cilium_bpf_map_ops_total{k8s_app="cilium"}[5m]) # Graph 확인

# 여러 시계열(metric series)의 값의 평균
avg(rate(cilium_bpf_map_ops_total{k8s_app="cilium"}[5m]))

# 집계 함수(예: sum, avg, max, rate)와 함께 사용하여 어떤 레이블(label)을 기준으로 그룹화할지를 지정하는 그룹핑(grouping) 
avg(rate(cilium_bpf_map_ops_total{k8s_app="cilium"}[5m])) by (pod)
avg(rate(cilium_bpf_map_ops_total{k8s_app="cilium"}[5m])) by (pod, map_name)
avg(rate(cilium_bpf_map_ops_total{k8s_app="cilium"}[5m])) by (pod, map_name, operation) # Graph 확인

# 시계열 중에서 가장 큰 k개를 선택
topk(5, avg(rate(cilium_bpf_map_ops_total{k8s_app="cilium"}[5m]))) by (pod, map_name, operation)
topk(5, avg(rate(cilium_bpf_map_ops_total{k8s_app="cilium", pod="cilium-4hghz"}[5m]))) by (pod, map_name, operation)
```

- **그라파나** 해당 대시보드 편집 → Variables
    - pod=~"`$pod`" 변수 사용
![[Pasted image 20250826150738.png]]

# 쿠버네티스 환경에서 속도 측정 테스트
## 배포 및 확인
```bash
# 배포
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iperf3-server
spec:
  selector:
    matchLabels:
      app: iperf3-server
  replicas: 1
  template:
    metadata:
      labels:
        app: iperf3-server
    spec:
      containers:
      - name: iperf3-server
        image: networkstatic/iperf3
        args: ["-s"]
        ports:
        - containerPort: 5201
---
apiVersion: v1
kind: Service
metadata:
  name: iperf3-server
spec:
  selector:
    app: iperf3-server
  ports:
    - name: tcp-service
      protocol: TCP
      port: 5201
      targetPort: 5201
    - name: udp-service
      protocol: UDP
      port: 5201
      targetPort: 5201
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iperf3-client
spec:
  selector:
    matchLabels:
      app: iperf3-client
  replicas: 1
  template:
    metadata:
      labels:
        app: iperf3-client
    spec:
      containers:
      - name: iperf3-client
        image: networkstatic/iperf3
        command: ["sleep"]
        args: ["infinity"]
EOF
```

```bash
# 확인 : 서버와 클라이언트가 어떤 노드에 배포되었는지 확인
kubectl get deploy,svc,pod -owide

# 서버 파드 로그 확인 : 기본 5201 포트 Listen
kubectl logs -l app=iperf3-server -f
```

## TCP 5201, 측정시간 5초
```bash
# 클라이언트 파드에서 아래 명령 실행
kubectl exec -it deploy/iperf3-client -- iperf3 -c iperf3-server -t 5

# 서버 파드 로그 확인 : 기본 5201 포트 Listen
kubectl logs -l app=iperf3-server -f
```

## UDP 사용, 역방향 모드(-R)
```bash
# 클라이언트 파드에서 아래 명령 실행
kubectl exec -it deploy/iperf3-client -- iperf3 -c iperf3-server -u -b 20G

# 서버 파드 로그 확인 : 기본 5201 포트 Listen
kubectl logs -l app=iperf3-server -f
```

## TCP, 쌍방향 모드(-R)
```bash
# 클라이언트 파드에서 아래 명령 실행
kubectl exec -it deploy/iperf3-client -- iperf3 -c iperf3-server -t 5 --bidir
...
[ ID][Role] Interval           Transfer     Bitrate         Retr
[  5][TX-C]   0.00-5.00   sec  31.2 GBytes  53.6 Gbits/sec   11             sender
[  5][TX-C]   0.00-5.00   sec  31.2 GBytes  53.6 Gbits/sec                  receiver
[  7][RX-C]   0.00-5.00   sec  23.2 GBytes  39.9 Gbits/sec   14             sender
[  7][RX-C]   0.00-5.00   sec  23.2 GBytes  39.8 Gbits/sec                  receiver
>> Client→Server (TX): 53.6 Gbps
>> Server→Client (RX): 39.9 Gbps
>> Retransmit : TX=11, RX=14 → 일부 패킷 손실로 인한 재전송이 있었음
>> 전체적으로는 40~54Gbps 수준의 대역폭이 측정됨

# 서버 파드 로그 확인 : 기본 5201 포트 Listen
kubectl logs -l app=iperf3-server -f
```

## TCP 다중 스트림(30개), -P(number of parallel client streams to run)
```bash
# 클라이언트 파드에서 아래 명령 실행
kubectl exec -it deploy/iperf3-client -- iperf3 -c iperf3-server -t 10 -P 2

# 서버 파드 로그 확인 : 기본 5201 포트 Listen
kubectl logs -l app=iperf3-server -f
```

삭제: kubectl delete deploy iperf3-server iperf3-client && kubectl delete svc iperf3-server


# cilium connectivity {test, perf} - [Docs](https://docs.cilium.io/en/stable/contributing/testing/e2e/)
- Cilium uses [cilium-cli connectivity tests](https://github.com/cilium/cilium-cli/#connectivity-check) for implementing and running end-to-end tests which test Cilium all the way from the API level (for example, importing policies, CLI) to the datapath (in order words, whether policy that is imported is enforced accordingly in the datapath).
## cilium connectivity test **: 기능 검증 - 기능별 시나리오(Pass , Fail)**
```bash
#
cilium connectivity -h
Connectivity troubleshooting
Available Commands:
  perf        Test network performance
  test        Validate connectivity in cluster

# 아래 test 실행 후 cilium-test-1 네임스페이스 접속
open http://127.0.0.1:30003/?namespace=cilium-test-1

# test : 테스트 항목 122개 2:59~
cilium connectivity test --debug  
🐛 [cilium-test-1] Registered connectivity tests
🐛   <Test no-policies, 8 scenarios, 0 resources, expectFunc <nil>>
🐛   <Test no-policies-from-outside, 1 scenarios, 0 resources, expectFunc <nil>>
🐛   <Test no-policies-extra, 2 scenarios, 0 resources, expectFunc <nil>>
🐛   <Test allow-all-except-world, 4 scenarios, 1 resources, expectFunc <nil>>
🐛   <Test client-ingress, 1 scenarios, 1 resources, expectFunc 0x104e81740>
🐛   <Test client-ingress-knp, 1 scenarios, 1 resources, expectFunc 0x104e81200>
🐛   <Test allow-all-with-metrics-check, 1 scenarios, 0 resources, expectFunc 0x104e628e0>
🐛   <Test all-ingress-deny, 2 scenarios, 1 resources, expectFunc 0x104e61750>
🐛   <Test all-ingress-deny-from-outside, 1 scenarios, 1 resources, expectFunc 0x104e61e90>
🐛   <Test all-ingress-deny-knp, 2 scenarios, 1 resources, expectFunc 0x104e62320>
🐛   <Test all-egress-deny, 2 scenarios, 1 resources, expectFunc 0x104e838e0>
🐛   <Test all-egress-deny-knp, 2 scenarios, 1 resources, expectFunc 0x104e83820>
🐛   <Test all-entities-deny, 2 scenarios, 1 resources, expectFunc 0x104e83760>
🐛   <Test cluster-entity, 1 scenarios, 1 resources, expectFunc 0x104e80d80>
🐛   <Test cluster-entity-multi-cluster, 1 scenarios, 1 resources, expectFunc 0x104e80cc0>
🐛   <Test host-entity-egress, 1 scenarios, 1 resources, expectFunc 0x104e7ff90>
🐛   <Test host-entity-ingress, 1 scenarios, 1 resources, expectFunc <nil>>
...
...
🐛   <Test pod-to-pod-no-frag, 1 scenarios, 0 resources, expectFunc <nil>>
🐛   <Test seq-bgp-control-plane-v1, 1 scenarios, 0 resources, expectFunc <nil>>
🐛   <Test seq-bgp-control-plane-v2, 1 scenarios, 0 resources, expectFunc <nil>>
🐛   <Test multicast, 1 scenarios, 0 resources, expectFunc <nil>>
🐛   <Test strict-mode-encryption, 1 scenarios, 0 resources, expectFunc 0x104e7f460>
🐛   <Test strict-mode-encryption-v2, 1 scenarios, 0 resources, expectFunc 0x104e7f520>
🐛   <Test host-firewall-ingress, 1 scenarios, 1 resources, expectFunc 0x104e7fc80>
🐛   <Test host-firewall-egress, 1 scenarios, 1 resources, expectFunc 0x104e7fe30>
🐛   <Test seq-client-egress-l7-tls-deny-without-headers, 1 scenarios, 2 resources, expectFunc 0x104e82b10>
🐛   <Test seq-client-egress-l7-tls-headers, 1 scenarios, 2 resources, expectFunc 0x104e82a50>
🐛   <Test seq-client-egress-l7-extra-tls-headers, 1 scenarios, 2 resources, expectFunc 0x104e82990>
🐛   <Test seq-client-egress-l7-tls-headers-port-range, 1 scenarios, 2 resources, expectFunc 0x104e82a50>
🐛   <Test no-unexpected-packet-drops, 1 scenarios, 0 resources, expectFun <nil>
...

# 실습 리소스 삭제
kubectl delete ns cilium-test-1
```

## cilium connectivity perf : 성능 측정(Gbsp, Latency) - Throughput, Retransmit, Latency
```bash
#
cilium connectivity perf -h
Flags:
      --bandwidth                        Test pod network bandwidth manage
      --crr                              Run CRR test
  -d, --debug                            Show debug messages
      --duration duration                Duration for the Performance test to run (default 10s)
  -h, --help                             help for perf
      --host-net                         Test host network (default true)
      --host-to-pod                      Test host-to-pod traffic
      --msg-size int                     Size of message to use in UDP test (default 1024)
      --namespace-labels map             Add labels to the connectivity test namespace
      --net-qos                          Test pod network Quality of Service
      --node-selector-client map         Node selector for the other-node client pod
      --node-selector-server map         Node selector for the server pod (and client same-node)
      --other-node                       Run tests in which the client and the server are hosted on difference nodes (default true)
      --performance-image string         Image path to use for performance (default "quay.io/cilium/network-perf:1751527436-c2462ae@sha256:0c491ed7ca63e6c526593b3a2d478f856410a50fbbce7fe2b64283c3015d752f")
      --pod-net                          Test pod network (default true)
      --pod-to-host                      Test pod-to-host traffic
      --print-image-artifacts            Prints the used image artifacts
      --report-dir string                Directory to save perf results in json format
      --rr                               Run RR test (default true)
      --same-node                        Run tests in which the client and the server are hosted on the same node (default true)
      --samples int                      Number of Performance samples to capture (how many times to run each test) (default 1)
      --setup-delay duration             Extra delay before starting the performance tests
      --streams uint                     The parallelism of tests with multiple streams (default 4)
      --test-namespace string            Namespace to perform the connectivity in (always suffixed with a sequence number to be compliant with test-concurrency param, e.g.: cilium-test-1) (default "cilium-test")
      --throughput                       Run throughput test (default true)
      --throughput-multi                 Run throughput test with multiple streams (default true)
      --tolerations strings              Extra NoSchedule tolerations added to test pods
      --udp                              Run UDP tests
      --unsafe-capture-kernel-profiles   Capture kernel profiles during test execution. Warning: run on disposable nodes only, as it installs additional software and modifies their configuration
```

