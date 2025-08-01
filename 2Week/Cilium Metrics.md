- eCHO Episode 170: Cilium Metrics Review - [Link](https://www.youtube.com/watch?v=aOm1YUO2Vmo)
- Cilium Metrics 설정 및 수집 방법 - [Docs](https://docs.cilium.io/en/stable/observability/metrics/#cilium-metrics)
	- Cilium metrics은 Cilium 자체의 상태, 즉 Cilium Agent, Cilium envoy, Cilium operator 프로세스에 대한 인사이트를 제공합니다.
	- 프로메테우스 메트릭이 활성화된 Cilium을 실행하려면 `prometheus.enabled=true` Helm 값 집합을 사용하여 Cilium을 배포하세요.
	- Cilium metrics are exported under the `cilium_` Prometheus namespace.
	- Envoy metrics are exported under the `envoy_` Prometheus namespace, of which the Cilium-defined metrics are exported under the `envoy_cilium_` namespace.
	- Kubernetes에서 실행 및 수집할 때 포드 이름과 네임스페이스로 태그가 지정됩니다.
	- 설정 → 이미 되어 있음
```bash
#
helm install cilium cilium/cilium --version 1.17.6 \\
  --namespace kube-system \\
  --set prometheus.enabled=true \\
  --set operator.prometheus.enabled=true

# The ports can be configured via prometheus.port, envoy.prometheus.port, or operator.prometheus.port respectively.
--set prometheus.port
--set envoy.prometheus.port
--set operator.prometheus.port
...
```
- 메트릭이 활성화되면 모든 Cilium 구성 요소에는 다음과 같은 주석이 표시됩니다. 주석은 Prometheus에게 메트릭을 스크랩할지 여부를 알리는 데 사용합니다.

```bash
# cilium-agent 데몬셋 파드
kubectl describe pod -n kube-system -l k8s-app=cilium | grep prometheus
					  prometheus.io/port: 9962
					  prometheus.io/scrape: true
curl 192.168.10.100:9962/metrics

# cilium-operator 디플로이먼트 파드 
kubectl describe pod -n kube-system -l name=cilium-operator | grep prometheus
Annotations:          prometheus.io/port: 9963
					  prometheus.io/scrape: true
curl 192.168.10.100:9963/metrics
```

- This additional headless service in addition to the other Cilium components is needed as each component can only have one Prometheus scrape and port annotation.
![[스크린샷 2025-07-27 오전 1.54.54.png]]

- Prometheus will pick up the Cilium and Envoy metrics automatically if the following option is set in the `scrape_configs` section:
        
```bash
#
kc describe cm -n cilium-monitoring prometheus
...
# <https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml#L156>
  - job_name: 'kubernetes-pods'
	kubernetes_sd_configs:
	  - role: pod
	relabel_configs:
	  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
		action: keep
		regex: true
	  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
		action: replace
		target_label: __metrics_path__
		regex: (.+)
	  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
		action: replace
		regex: (.+):(?:\\d+);(\\d+)
		replacement: ${1}:${2}
		target_label: __address__
	  - action: labelmap
		regex: __meta_kubernetes_pod_label_(.+)
	  - source_labels: [__meta_kubernetes_namespace]
		action: replace
		target_label: namespace
	  - source_labels: [__meta_kubernetes_pod_name]
		action: replace
		target_label: pod
	  - source_labels: [__meta_kubernetes_pod_container_port_number]
		action: keep
		regex: \\d+
```
        
- Hubble Metrics 설정 및 수집 방법 - [Docs](https://docs.cilium.io/en/stable/observability/metrics/#hubble-metrics)
    - Cilium 메트릭은 Cilium 상태 자체를 모니터링할 수 있게 해주지만, Hubble 메트릭은 Cilium이 관리하는 쿠버네티스 포드의 네트워크 동작을 연결 및 보안과 관련하여 모니터링할 수 있게 해줍니다.
    - 설정 → 이미 되어 있음
        - Some of the metrics can also be configured with additional options. See the [Hubble exported metrics](https://docs.cilium.io/en/stable/observability/metrics/#hubble-exported-metrics) section for the full list of available metrics and their options.
```bash
#
helm install cilium cilium/cilium --version 1.17.6 \\
  --namespace kube-system \\
  --set prometheus.enabled=true \\
  --set operator.prometheus.enabled=true \\
  --set hubble.enabled=true \\
  --set hubble.metrics.enableOpenMetrics=true \\
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\\,source_namespace\\,source_workload\\,destination_ip\\,destination_namespace\\,destination_workload\\,traffic_direction}"
  --set hubble.metrics.port
```

L7 metrics such as HTTP, are only emitted for pods that enable [Layer 7 Protocol Visibility](https://docs.cilium.io/en/stable/observability/visibility/#proxy-visibility). ⇒ L7 메트릭은 L7 가시성 활성화 필요!

- When deployed with a non-empty `hubble.metrics.enabled` Helm value, the Cilium chart will create a Kubernetes headless service named `hubble-metrics` with the `prometheus.io/scrape:'true'` annotation set:
        
```bash
# hubble-metrics 헤드리스 서비스 정보 확인
kubectl get svc -n kube-system hubble-metrics
kc describe svc -n kube-system hubble-metrics
...
	  prometheus.io/port: 9965
	  prometheus.io/scrape: true
...
Endpoints:                192.168.10.101:9965,192.168.10.100:9965,192.168.10.102:9965
...

curl 192.168.10.100:9965/metrics

#
kc describe cm -n cilium-monitoring prometheus
  # <https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml#L79>
  - job_name: 'kubernetes-endpoints'
	kubernetes_sd_configs:
	  - role: endpoints
	relabel_configs:
	  - source_labels: [__meta_kubernetes_pod_label_k8s_app]
		action: keep
		regex: cilium
	  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
		action: keep
		regex: true
	  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
		action: replace
		target_label: __scheme__
		regex: (https?)
	  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
		action: replace
		target_label: __metrics_path__
		regex: (.+)
	  - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
		action: replace
		target_label: __address__
		regex: (.+)(?::\\d+);(\\d+)
		replacement: $1:$2
	  - action: labelmap
		regex: __meta_kubernetes_service_label_(.+)
	  - source_labels: [__meta_kubernetes_namespace]
		action: replace
		target_label: namespace
	  - source_labels: [__meta_kubernetes_service_name]
		action: replace
		target_label: service
```
![[Pasted image 20250727020025.png]]

- OpenMetrics - [Docs](https://docs.cilium.io/en/stable/observability/metrics/#openmetrics) , OpenMetrics - [Home](https://openmetrics.io/) , [Github](https://github.com/prometheus/OpenMetrics) , [Spec](https://prometheus.io/docs/specs/om/open_metrics_spec/) , [https://meetup.nhncloud.com/posts/284](https://meetup.nhncloud.com/posts/284)
    - `hubble.metrics.enableOpenMetrics=true`를 설정하여 OpenMetrics에 opt-in할 수 있습니다.
    - OpenMetrics를 활성화하면 클라이언트가 명시적으로 요청할 때 Hubble metrics endpoint가 OpenMetrics 형식으로 메트릭 내보내기를 지원하도록 구성됩니다.
    - OpenMetrics를 사용하면 Export된 메트릭에 트레이스 ID를 삽입하여 메트릭을 트레이스와 연결할 수 있는 Examplears와 같은 추가 기능을 지원합니다.
    - Prometheus는 OpenMetrics를 활용하도록 구성해야 하며, Exemplar storage이 활성화된 경우에만 예제를 스크랩합니다. - [Docs](https://prometheus.io/docs/prometheus/latest/feature_flags/#exemplars-storage)
    - OpenMetrics는 메트릭 이름과 레이블에 몇 가지 추가 요구 사항을 부과하므로 현재 이 기능은 opt-in 상태입니다. 그러나 우리는 모든 허블 메트릭이 OpenMetrics 요구 사항을 준수한다고 믿습니다.

- Cluster Mesh API Server Metrics : Skip - [Docs](https://docs.cilium.io/en/stable/observability/metrics/#cluster-mesh-api-server-metrics)
- Metrics Reference : 개별 메트릭에 대한 설명 - [Docs](https://docs.cilium.io/en/stable/observability/metrics/#metrics-reference)
    - cilium-agent : - [Link](https://docs.cilium.io/en/stable/observability/metrics/#cilium-agent)
        - Feature Metrics : Advanced Connectivity and Load Balancing, Control Plane, Datapath, Network Policies
        - Exported Metrics : Endpoint, Service, Cluster health, Node Connectivity, Clustermesh, Datapath, IPSec, eBPF, Drops/Forwards(L3/L4)…
    - cilium-operator : - [Link](https://docs.cilium.io/en/stable/observability/metrics/#cilium-operator)
        - Feature Metrics : Advanced Connectivity and Load Balancing
        - Exported Metrics : BGP Control Operator, IPAM, LB-IPAM, Controllers, CiliumEndpointSlices, Unmanaged Pods..
    - Hubble : 다른 메트릭과 다르게 semicolon-separated options per metric 제공 - [Link](https://docs.cilium.io/en/stable/observability/metrics/#hubble)
        - 예시) `--hubble-metrics="dns:query;ignoreAAAA http:destinationContext=workload-name"` will enable the `dns` metric with the `query` and `ignoreAAAA` options, and the `http` metric with the `destinationContext=workload-name` option.
        - Context Options
            - `sourceContext` - Configures the `source` label on metrics for both egress and ingress traffic.
            - `sourceEgressContext` - Configures the `source` label on metrics for egress traffic (takes precedence over `sourceContext`).
            - `sourceIngressContext` - Configures the `source` label on metrics for ingress traffic (takes precedence over `sourceContext`).
            - `destinationContext` - Configures the `destination` label on metrics for both egress and ingress traffic.
            - `destinationEgressContext` - Configures the `destination` label on metrics for egress traffic (takes precedence over `destinationContext`).
            - `destinationIngressContext` - Configures the `destination` label on metrics for ingress traffic (takes precedence over `destinationContext`).
            - `labelsContext` - Configures a list of labels to be enabled on metrics.
        - Most Hubble metrics can be configured to add the source and/or destination context as a label using the `sourceContext` and `destinationContext` options. The possible values are:

|**Option Value**|**Description**|
|---|---|
|`identity`|All Cilium security identity labels|
|`namespace`|Kubernetes namespace name|
|`pod`|Kubernetes pod name and namespace name in the form of `namespace/pod`.|
|`pod-name`|Kubernetes pod name.|
|`dns`|All known DNS names of the source or destination (comma-separated)|
|`ip`|The IPv4 or IPv6 address|
|`reserved-identity`|Reserved identity label.|
|`workload`|Kubernetes pod’s workload name and namespace in the form of `namespace/workload-name`.|
|`workload-name`|Kubernetes pod’s workload name (workloads are: Deployment, Statefulset, Daemonset, ReplicationController, CronJob, Job, DeploymentConfig (OpenShift), etc).|
|`app`|Kubernetes pod’s app name, derived from pod labels (`app.kubernetes.io/name`, `k8s-app`, or `app`).|

- 나머지는 공식문서 내용 확인 할 것!