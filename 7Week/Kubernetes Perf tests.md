- [https://github.com/kubernetes/perf-tests](https://github.com/kubernetes/perf-tests) : Kubernetes-related performance test related tools
- [https://github.com/kubernetes/perf-tests/tree/master/util-images](https://github.com/kubernetes/perf-tests/tree/master/util-images) : Util, Images

# ClusterLoader2 (CL2) : Kubernetes 부하 테스트 도구이자 공식 K8s 확장성 및 성능 테스트 프레임워크 - [Link](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2)

- 테스트는 클러스터의 원하는 상태 집합(예: 10,000개의 포드, 2,000개의 클러스터 IP 서비스 또는 5개의 데몬 세트를 실행)을 정의하고 각 상태에 도달하는 속도(포드 처리량)를 지정합니다.
- CL2는 또한 측정할 성능 특성을 정의합니다( 자세한 내용은 [측정 목록](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2#Measurement) 참조 ).
- 마지막으로, CL2는 [Prometheus를](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2#prometheus-metrics) 사용한 테스트 중에 클러스터에 대한 추가적인 관측성을 제공합니다.

## 사용 가능한 측정값
- **APIAvailabilityMeasurement**
    이 측정값은 클러스터 제어 평면의 가용성에 대한 정보를 수집합니다.
    이를 측정하는 두 가지 방법이 약간 다릅니다.
    - 클러스터 수준 가용성, 여기서 우리는 주기적으로 API 호출을 발행합니다 `/readyz`
    - 호스트 수준 가용성은 각 제어 평면의 호스트 `/readyz` 엔드포인트를 주기적으로 폴링하는 것입니다.
        - 이렇게 하려면 [exec 서비스를](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2/pkg/execservice) 활성화해야 합니다.
- **API 응답성**
    PrometheusSimple 이 측정값은 Prometheus 서버에서 수집한 데이터를 기반으로 서버 API 호출의 지연 시간과 호출 횟수에 대한 백분위수를 생성합니다. API 호출은 리소스, 하위 리소스, 동사 및 범위로 구분됩니다.
    이 측정값은 [API 호출 지연 시간 SLO가](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/api_call_latency.md) 충족되는지 확인합니다. Prometheus 서버를 사용할 수 없는 경우 측정은 건너뜁니다.
- **APIResponsivenessPrometheus**
    이 측정값은 Prometheus 서버에서 수집한 데이터를 기반으로 서버 API 호출의 지연 시간과 개수에 대한 요약을 생성합니다. API 호출은 리소스, 하위 리소스, 동사, 범위로 구분됩니다. 이 측정값은 [API 호출 지연 시간 SLO가](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/api_call_latency.md) 충족되는지 확인합니다 . Prometheus 서버를 사용할 수 없는 경우 측정은 건너뜁니다.
- **CPUProfile**
    이 측정은 pprof가 제공한 특정 구성 요소에 대한 CPU 사용 프로필을 수집합니다.
- **EtcdMetrics**
    이 측정은 일련의 etcd 메트릭과 해당 데이터베이스 크기를 수집합니다.
- **MemoryProfile**
    이 측정은 pprof가 제공한 특정 구성 요소에 대한 메모리 프로필을 수집합니다.
- **MetricsForE2E**
    측정은 kube-apiserver, 컨트롤러 관리자, 스케줄러 및 선택적으로 모든 kubelet에서 메트릭을 수집합니다.
- **PodPeriodicCommand**
    이 측정은 레이블 선택기로 지정된 포드에서 일정 간격으로 명령을 지속적으로 실행합니다. 각 명령의 출력이 수집되어 측정 기간 동안 CPU 및 메모리 프로필과 같은 정보를 폴링할 수 있습니다.
- **PodStartupLatency**
    이 측정은 [Pod 시작 SLO가](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/pod_startup_latency.md) 충족되는지 확인합니다.
- **ResourceUsageSummary:**
    이 측정은 구성 요소별 리소스 사용량을 수집합니다. 수집 실행 중 수집된 데이터는 관찰된 각 구성 요소의 90, 99, 100번째 사용 백분위수를 나타내는 요약으로 변환됩니다.
    선택적으로 리소스 제약 조건 파일을 측정에 제공할 수 있습니다. 리소스 제약 조건 파일은 특정 구성 요소의 CPU 및/또는 메모리 제약 조건을 지정합니다. 제약 조건 중 하나라도 위반하면 오류가 반환되고 테스트가 실패합니다.
- **SchedulingMetrics**
    이 측정은 스케줄러 메트릭 세트를 수집합니다.
- **SchedulingThroughput**
    이 측정은 스케줄링 처리량을 수집합니다.
- **타이머**
    타이머를 사용하면 테스트의 특정 부분에 대한 대기 시간을 측정할 수 있습니다(단일 타이머로 다양한 작업을 독립적으로 측정할 수 있음).
- **WaitForControlledPodsRunning**
    이 측정값은 지정된 제어 객체(ReplicationController, ReplicaSet, Deployment, DaemonSet 및 Job)가 모든 포드를 실행할 때까지 대기하는 장벽 역할을 합니다. 제어 객체는 레이블 선택자, 필드 선택자 및 네임스페이스로 지정할 수 있습니다. 시간 초과가 발생하면 테스트는 계속 실행되고 오류가 기록되며 테스트는 실패로 표시됩니다.
- **WaitForRunningPods:**
    필요한 수의 Pod가 실행될 때까지 기다리는 배리어입니다. Pod는 레이블 선택자, 필드 선택자 및 네임스페이스로 지정할 수 있습니다. 시간 초과가 발생하면 테스트는 계속 실행되고 오류가 기록되며 테스트는 실패로 표시됩니다.
- **Sleep**
    요청된 시간이 지날 때까지 기다리는 장벽입니다.
- **WaitForGenericK8sObjects**`Type=StatusNodeReady=True`
    는 필요한 개수의 K8s 객체가 필요한 조건을 충족할 때까지 대기하는 배리어입니다. 이러한 조건은
    다음과 같이 형식 요구 사항 목록으로 지정할 수 있습니다
    . 시간 초과가 발생하면 테스트는 계속 실행되고 오류가 기록되며 테스트는 실패로 표시됩니다.

- kind 에서 1대 노드에서 간단 테스트 가이드 소개 - [Guide](https://github.com/kubernetes/perf-tests/blob/master/clusterloader2/docs/GETTING_STARTED.md)
- 100노드 규모 테스트 실행 - [Guide](https://github.com/kubernetes/perf-tests/blob/master/clusterloader2/docs/GETTING_STARTED.md) , [LoadCode](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2/testing/load)
    - 테스트 단계 : 객체 생성 → 원래 크기의 50%, 150% 사이로 개체 크기 조정 → 객체 삭제
    - **100개 노드**부터 **최대 5,000개 노드**까지 **클러스터를 테스트**하는 데 사용할 수 있습니다.
    - 부하 테스트는 약 **30개의 노드로 구성된 포드 객체를 생성합**니다. 다음과 같은 객체가 생성됩니다.
        - deployments
        - jobs
        - statefulsets
        - services
        - secrets
        - configmaps
- deployments, jobs and statefulsets - 소규모(포드 5개), 중규모(포드 30개), 대규모(포드 250개) 버전.
- First, you need to create 100-nodes cluster then you can run cluster scale test with this command:

```bash
# --enable-prometheus-server=true deploys prometheus server using prometheus-operator. 
./run-e2e-with-prometheus-fw-rule.sh cluster-loader2 --testconfig=./testing/load/config.yaml --nodes=100 --provider=gke --enable-prometheus-server=true --kubeconfig=${HOME}/.kube/config --v=2
```

- 프로메테우스 측정 기준에 따라 다양한 측정이 가능합니다. 예를 들어,
    - API 응답성 - kube-apiserver에 대한 요청 지연 시간을 측정합니다.
    - 스케줄링 처리량
    - NodeLocalDNS 대기 시간

# DNS (CoreDNS, nodeLocal DNSCache) : DNS 성능 테스트를 실행하는 데 사용되는 스크립트 포함 - [Link](https://github.com/kubernetes/perf-tests/tree/master/dns)
- The performance script `run` benchmarks the performance of a single DNS server instance with a synthetic query workload.
- Benchmarking the cluster DNS : coredns 성능 측정 - [결과](https://perf-dash.k8s.io/#/?jobname=kube-dns%20benchmark&metriccategoryname=dns&metricname=Latency&query_file=all-queries.txt&run_length_seconds=600)
- Comparing cluster DNS and NodeLocal DNSCache : NodeLocal DNSCache 활성화 시 성능 비교 - [결과](https://perf-dash.k8s.io/#/?jobname=node-local-dns%20benchmark&metriccategoryname=dns&metricname=Latency&query_file=all-queries.txt&run_length_seconds=600)
- CoreDNS and kube-dns v1.5+ (image `registry.k8s.io/kubedns-amd64:1.9`) can export [Prometheus](http://prometheus.io/) metrics.
- 성능 측정 관련 질문?
    - What is the maximum queries per second (**QPS**) we can get from the **Kubernetes DNS service** given no limits?
    - If we **restrict CPU resources,** what is the **performance** we can expect? (i.e. resource limits in the pod yaml).
    - What are the SLOs (e.g. query latency) for a given setting that the user can expect? Alternate phrasing: what can we expect in realistic workloads that do not saturate the service?

# Benchmarking Kubernetes Networking Performance : Kubernetes 네트워킹 성능을 측정하는 표준화된 벤치마크 - [Link](https://github.com/kubernetes/perf-tests/tree/master/network/benchmarks/netperf)

- 벤치마크는 단일 Go 바이너리 호출을 통해 실행할 수 있으며, 아래에서 볼 수 있듯이 오케스트레이터 및 워커 포드에 있는 모든 자동화 테스트를 트리거합니다.
- 테스트는 go 바이너리와 iperf3 및 기타 도구가 내장된 사용자 지정 도커 컨테이너를 사용합니다.
- 오케스트레이터 포드는 아래 설명된 5가지 시나리오에 대해 MTU(TCP의 경우 MSS 튜닝, UDP의 경우 직접 패킷 크기 튜닝)를 사용하여 직렬 순서로 테스트를 실행하도록 워커 포드를 조정합니다.
- 노드 레이블을 사용하여 워커 포드 1과 2는 동일한 쿠버네티스 노드에 배치되고, 워커 포드 3은 다른 노드에 배치됩니다.
- 모든 노드는 간단한 golang rpcs를 사용하여 오케스트레이터 포드 서비스와 통신하고 작업 항목을 요청합니다.
- 이 테스트에는 최소 두 개의 쿠버네티스 워커 노드가 필요합니다.

```bash
The 5 major network traffic paths are combination of Pod IP vs Virtual IP and whether the pods are co-located on the same node/VM versus a remotely located pod.
> Same VM using Pod IP

Same VM Pod to Pod traffic tests from Worker 1 to Worker 2 using its Pod IP.
> Same VM using Cluster/Virtual IP

Same VM Pod to Pod traffic tests from Worker 1 to Worker 2 using its Service IP (also known as its Cluster IP or Virtual IP).
> Remote VM using Pod IP

Worker 3 to Worker 2 traffic tests using Worker 2 Pod IP.
> Remote VM using Cluster/Virtual IP

Worker 3 to Worker 2 traffic tests using Worker 2 Cluster/Virtual IP.
> Same VM Pod Hairpin to itself using Cluster IP
```

- 오케스트레이터와 워커 파드는 이니시에이터 스크립트와 독립적으로 실행되며, 오케스트레이터 파드는 테스트 케이스 일정이 완료될 때까지 워커들에게 작업 항목을 전송합니다.
- 모든 워커 노드의 **iperf** 출력(TCP 및 UDP 모드 모두)과 **netperf TCP** 출력은 오케스트레이터 파드에 업로드되어 필터링되고 그 결과는 netperf.csv에 기록됩니다.
- 그런 다음 실행 스크립트는 netperf.csv 파일을 추출하여 로컬 디스크에 기록합니다. csv 파일의 모든 단위는 Mbit/초입니다.
- 모든 쿠버네티스 엔티티는 "netperf" 네임스페이스 아래에 생성됩니다.
- CSV 데이터 출력
```bash
MSS, Maximum, 96, 352, 608, 864, 1120, 1376,
1 iperf TCP. Same VM using Pod IP            ,35507.000000,33835,33430,35372,35220,35373,35507,
2 iperf TCP. Same VM using Virtual IP        ,32997.000000,32689,32997,32256,31995,31904,31830,
3 iperf TCP. Remote VM using Pod IP          ,10652.000000,8793,9836,10602,9959,9941,10652,
4 iperf TCP. Remote VM using Virtual IP      ,11046.000000,10429,11046,10064,10622,10528,10246,
5 iperf TCP. Hairpin Pod to own Virtual IP   ,32400.000000,31473,30253,32075,32058,32400,31734,
6 iperf UDP. Same VM using Pod IP            ,10642.000000,10642,
7 iperf UDP. Same VM using Virtual IP        ,8983.000000,8983,
8 iperf UDP. Remote VM using Pod IP          ,11143.000000,11143,
9 iperf UDP. Remote VM using Virtual IP      ,10836.000000,10836,
10 netperf. Same VM using Pod IP             ,11675.380000,11675.38,
11 netperf. Same VM using Virtual IP         ,0.000000,0.00,
12 netperf. Remote VM using Pod IP           ,6646.820000,6646.82,
13 netperf. Remote VM using Virtual IP       ,0.000000,0.00,
```

- CSV 데이터 그래프화
    - **matplotlib**을 사용하여 csv 데이터(및 호환성을 위해 PNG 및 JPG 파일)에서 그래프 SVG 파일을 생성합니다.
        - 파이썬 Matplotlib 라인 차트
        - 막대 차트
    - MSS를 무시하고 최대 대역폭 수치를 사용하면 막대형 차트가 성능을 비교하는 데 더 나은 도구가 될 수 있습니다.
    - CSV 데이터를 **Google 시트**로 가져와서 **그래프**로 표현하는 것도 가능합니다.

# Network policy enforcement latency : Pod 및 네트워크 정책 변경에 대한 네트워크 정책 적용 지연 시간을 측정 - [Link](https://github.com/kubernetes/perf-tests/tree/master/network/tools/network-policy-enforcement-latency)

# Kubernetes Perfdash : 성능 지표를 수집하고 표시하는 웹 UI - [Link](https://github.com/kubernetes/perf-tests/tree/master/perfdash) , [Demo](https://perf-dash.k8s.io/#/?jobname=aws-100Nodes&metriccategoryname=APIServer&metricname=LoadInitEventsCount&resource=configmaps&group=)
- 성능 지표는 다양한 노드 수, 플랫폼 유형 및 플랫폼 버전에 대한 성능 테스트 결과를 기반으로 생성됩니다.
- 지원되는 메트릭
    - Responsiveness
    - Resources
    - PodStartup
    - TestPhaseTimer
    - RequestCount
    - RequestCountByClient

# Kubernetes Performance SLO monitor : api 서버로 수집할 수 없는 성능 SLOs 를 모니터링 및 메트릭 노출 - [Link](https://github.com/kubernetes/perf-tests/tree/master/slo-monitor)
- SLO monitor is a simple pod that needs to be able to read Pods and Events.
- On top of that it should work as long as it can communicate with the API server.

# 도전과제4 perfdash 설치로 성능 지표 수집해보기

^47f260

# 도전과제5 perf-tests 테스트 시 메트릭 노출 후 프로메테우스/그라파나 수집 분석 해보기

^c57180
