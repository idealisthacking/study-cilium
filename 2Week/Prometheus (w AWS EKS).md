
> [!note]  하기 내용은 AWS EKS에 배포된 것으로 UI가 다를 수 있음.

```bash
# 아래 처럼 프로메테우스가 각 서비스의 포트 접속하여 메트릭 정보를 수집
kubectl get node -owide
kubectl get svc,ep -n monitoring kube-prometheus-stack-prometheus-node-exporter

# (노드 익스포터 경우) 노드의 9100번 포트의 /metrics 접속 시 다양한 메트릭 정보를 확인할수 있음 : 마스터 이외에 워커노드도 확인 가능
ssh ec2-user@$N1 curl -s localhost:9100/metrics
```
    
- 프로메테우스 ingress 도메인으로 웹 접속
    
```bash
# ingress 확인
kubectl get ingress -n monitoring kube-prometheus-stack-prometheus
kubectl describe ingress -n monitoring kube-prometheus-stack-prometheus

# 프로메테우스 ingress 도메인으로 웹 접속
echo -e "Prometheus Web URL = https://prometheus.$MyDomain"
open "https://prometheus.$MyDomain" macOS

# 웹 상단 주요 메뉴 설명
1. 쿼리(Query) : 프로메테우스 자체 검색 언어 PromQL을 이용하여 메트릭 정보를 조회 -> 단순한 그래프 형태 조회
2. 경고(Alerts) : 사전에 정의한 시스템 경고 정책(Prometheus Rules)에 대한 상황
3. 상태(Status) : 경고 메시지 정책(Rules), 모니터링 대상(Targets) 등 다양한 프로메테우스 설정 내역을 확인 > 버전 정보
```

- 쿼리 입력 옵션
    ![[Pasted image 20250727012329.png]]
	- Use local time : 출력 시간을 로컬 타임으로 변경
	- Enable query history : PromQL 쿼리 히스토리 활성화
	- Enable autocomplete : 자동 완성 기능 활성화
	- Enable highlighting : 하이라이팅 기능 활성화
	- Enable linter : 문법 오류 감지, 자동 코스 스타일 체크
        
- Statues → 프로메테우스 설정(Configuration) 확인 : Status → Runtime & Build Information 클릭
	- Storage retention : 5d or 10GiB → 메트릭 저장 기간이 5일 경과 혹은 10GiB 이상 시 오래된 것부터 삭제 ⇒ helm 파라미터에서 수정 가능
- Statues → 프로메테우스 설정(Configuration) 확인 : Status → Command-Line Flags 클릭
	- -log.level : info
	- -storage.tsdb.retention.size : 10GiB
	- -storage.tsdb.retention.time : 5d
- Statues → 프로메테우스 설정(Configuration) 확인 : Status → Configuration

![[Pasted image 20250727012457.png]]
https://prometheus.io/docs/introduction/overview/

- job name 을 기준으로 scraping

```bash
global:
  scrape_interval: 15s     # 메트릭 가져오는(scrape) 주기
  scrape_timeout: 10s      # 메트릭 가져오는(scrape) 타임아웃
  evaluation_interval: 15s # alert 보낼지 말지 판단하는 주기
...
- job_name: serviceMonitor/monitoring/kube-prometheus-stack-prometheus-node-exporter/0
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  ...
  relabel_configs:
  - source_labels: [job]
	separator: ;
	target_label: __tmp_prometheus_job_name
	replacement: $1
	action: replace
  - source_labels: [__meta_kubernetes_service_label_app_kubernetes_io_instance, __meta_kubernetes_service_labelpresent_app_kubernetes_io_instance]
	separator: ;
	regex: (kube-prometheus-stack);true
	replacement: $1
	action: keep
  - source_labels: [__meta_kubernetes_service_label_app_kubernetes_io_name, __meta_kubernetes_service_labelpresent_app_kubernetes_io_name]
	separator: ;
	regex: (prometheus-node-exporter);true
	replacement: $1
	action: keep
  ...
kubernetes_sd_configs:    # 서비스 디스커버리(SD) 방식을 이용하고, 파드의 엔드포인트 List 자동 반영
  - role: endpoints       # 서비스에 연결된 엔드포인트(Pod IP + Port) 탐색
	kubeconfig_file: ""   # Prometheus가 실행 중인 환경의 기본 kubeconfig 사용
	follow_redirects: true # 엔드포인트를 변경할 경우 이를 따라감
	enable_http2: true
	namespaces:
	  own_namespace: false # 자신이 실행 중인 네임스페이스가 아닌 곳에서도 탐색 가능
	  names:
	  - monitoring        # 서비스 엔드포인트가 속한 네임 스페이스 이름을 지정 : monitoring 네임스페이스에 있는 서비스만 타겟팅, 서비스 네임스페이스가 속한 포트 번호를 구분하여 메트릭 정보를 가져옴
...

- job_name: podMonitor/kube-system/aws-cni-metrics/0
  honor_timestamps: true
  ...
  relabel_configs:
  - source_labels: [job] 
	separator: ;
	target_label: __tmp_prometheus_job_name
	replacement: $1
	action: replace # job 라벨 값을 __tmp_prometheus_job_name에 저장
  - source_labels: [__meta_kubernetes_pod_label_k8s_app, __meta_kubernetes_pod_labelpresent_k8s_app]
	separator: ;
	regex: (aws-node);true
	replacement: $1
	action: keep    # Pod의 k8s_app 라벨 값이 aws-node인 경우만 유지
  ...
kubernetes_sd_configs:
- role: pod               # 클러스터 내 모든 개별 Pod 탐색
  kubeconfig_file: ""
  follow_redirects: true
  enable_http2: true
  namespaces:
	own_namespace: false
	names:
	- kube-system
...
```

- 전체 메트릭 대상(Targets) 확인 : Status → Target health
	- 해당 스택은 ‘노드-익스포터’, cAdvisor, 쿠버네티스 전반적인 현황 이외에 다양한 메트릭을 포함
	- 현재 각 Target 클릭 시 메트릭 정보 확인 : 아래 예시

```bash
# serviceMonitor/monitoring/kube-prometheus-stack-kube-proxy/0 (3/3 up) 중 노드1에 Endpoint 접속 확인 (접속 주소는 실습 환경에 따라 다름)
ssh $N1 curl -s http://localhost:10249/metrics
rest_client_response_size_bytes_bucket{host="006fc3f3f0730a7fb3fdb3181f546281.gr7.ap-northeast-2.eks.amazonaws.com",verb="POST",le="4.194304e+06"} 1
rest_client_response_size_bytes_bucket{host="006fc3f3f0730a7fb3fdb3181f546281.gr7.ap-northeast-2.eks.amazonaws.com",verb="POST",le="1.6777216e+07"} 1
rest_client_response_size_bytes_bucket{host="006fc3f3f0730a7fb3fdb3181f546281.gr7.ap-northeast-2.eks.amazonaws.com",verb="POST",le="+Inf"} 1
rest_client_response_size_bytes_sum{host="006fc3f3f0730a7fb3fdb3181f546281.gr7.ap-northeast-2.eks.amazonaws.com",verb="POST"} 626
rest_client_response_size_bytes_count{host="006fc3f3f0730a7fb3fdb3181f546281.gr7.ap-northeast-2.eks.amazonaws.com",verb="POST"} 1
...

# [운영서버 EC2] serviceMonitor/monitoring/kube-prometheus-stack-api-server/0 (2/2 up) 중 Endpoint 접속 확인 (접속 주소는 실습 환경에 따라 다름) 
>> 해당 IP주소는 어디인가요?, 왜 apiserver endpoint는 2개뿐인가요? , 아래 메트릭 수집이 되게 하기 위해서는 어떻게 하면 될까요?
curl -s https://192.168.1.53/metrics | tail -n 5
...

# [운영서버 EC2] 그외 다른 타켓의 Endpoint 로 접속 확인 가능 : 예시) 아래는 coredns 의 Endpoint 주소 (접속 주소는 실습 환경에 따라 다름)
curl -s http://192.168.1.75:9153/metrics | tail -n 5
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 7.79350016e+08
# HELP process_virtual_memory_max_bytes Maximum amount of virtual memory available in bytes.
# TYPE process_virtual_memory_max_bytes gauge
process_virtual_memory_max_bytes 1.8446744073709552e+19
```
![[Pasted image 20250727012704.png]]

- 프로메테우스 설정(Configuration) 확인 : Status → Service Discovery : 모든 endpoint 로 도달 가능 시 자동 발견!, 도달 규칙은 설정Configuration 파일에 정의
	- 예) serviceMonitor/monitoring/kube-prometheus-stack-apiserver/0 경우 해당 __address__="*192.168.1.53*:443" 도달 가능 시 자동 발견됨

- 메트릭을 그래프(Graph)로 조회 : Graph - 아래 PromQL 쿼리(전체 클러스터 노드의 CPU 사용량 합계)입력 후 조회 → Graph 확인
	- 혹은 지구 아이콘(Metrics Explorer) 클릭 시 전체 메트릭 출력되며, 해당 메트릭 클릭해서 확인
        
```bash
node_cpu_seconds_total
node_cpu_seconds_total{mode="idle"}
(node_cpu_seconds_total{mode="idle"}[1m])
```
        
```bash
# 노드 메트릭
node 입력 후 자동 출력되는 메트릭 확인 후 선택
node_boot_time_seconds

# kube 메트릭
kube 입력 후 자동 출력되는 메트릭 확인 후 선택
```

- 프로메테우스 쿼리 : node-exporter , kube-state-metrics , kube-proxy
    - node-exporter : /proc, /sys 정보  - [Link](https://github.com/prometheus/node_exporter) [Docs](https://prometheus.io/docs/guides/node-exporter/)
        
![[Pasted image 20250727012801.png]]
https://www.opsramp.com/guides/prometheus-monitoring/prometheus-node-exporter/
        
```bash
# Table 아래 쿼리 입력 후 Execute 클릭 -> Graph 확인
## 출력되는 메트릭 정보는 node-exporter 를 통해서 노드에서 수집된 정보
node_memory_Active_bytes

# 특정 노드(인스턴스) 필터링 : 아래 IP는 출력되는 자신의 인스턴스 PrivateIP 입력 후 Execute 클릭 -> Graph 확인
node_memory_Active_bytes{instance="*192.168.1.105*:9100"}
```
![[Pasted image 20250727012836.png]]

- kube-state-metrics (ksm) : k8s api 통해 k8s 오브젝트 정보 수집 - [Link](https://github.com/kubernetes/kube-state-metrics)
![[Pasted image 20250727012907.png]]
https://medium.com/@seifeddinerajhi/monitoring-kubernetes-clusters-with-kube-state-metrics-2b9e73a67895
        
```bash
# replicas's number
kube_deployment_status_replicas
kube_deployment_status_replicas_available
kube_deployment_status_replicas_available{deployment="coredns"}

# scale out
kubectl scale deployment -n kube-system coredns --replicas 3

# 확인
kube_deployment_status_replicas_available{deployment="coredns"}

# scale in
kubectl scale deployment -n kube-system coredns --replicas 1
```
        
- kube-proxy :  [Github](https://github.com/kubernetes/kubernetes/blob/de7708f06e11efe1140805f1eb4814f358a8d31e/pkg/proxy/metrics/metrics.go) → 이미 애플리케이션 내장으로 메트릭 노출 준비 설정됨! (아래 코드 설명 by ChatGPT) → coredns 도? https://coredns.io/plugins/metrics/
	- Kubernetes의 kube-proxy에서 사용되는 성능 및 상태 모니터링을 위한 메트릭을 정의하는 코드이다.
	- iptables, IPVS, NFTables 등 프록시 모드별로 적절한 메트릭을 등록 및 관리한다.
	- Netfilter 기반의 패킷 통계(nfacct)도 지원하여 패킷 드롭 및 로컬 트래픽 모니터링 기능을 포함한다.
        
```bash
#
kubeproxy_sync_proxy_rules_iptables_total
kubeproxy_sync_proxy_rules_iptables_total{table="filter"}
kubeproxy_sync_proxy_rules_iptables_total{table="nat"}
kubeproxy_sync_proxy_rules_iptables_total{table="nat", instance="192.168.1.188:10249"}
```

- 프로메테우스 쿼리 : 애플리케이션 - NGINX 웹서버 애플리케이션 모니터링 설정 및 접속
    - 서비스모니터 동작

![[Pasted image 20250727013021.png]]
https://containerjournal.com/topics/container-management/cluster-monitoring-with-prometheus-operator/
        
- nginx 를 helm 설치 시 프로메테우스 익스포터 Exporter 옵션 설정 시 자동으로 nginx 를 프로메테우스 모니터링에 등록 가능!
	- 프로메테우스 설정에서 nginx 모니터링 관련 내용을 서비스 모니터 CRD로 추가 가능!
- 기존 애플리케이션 파드에 프로메테우스 모니터링을 추가하려면 사이드카 방식을 사용하며 exporter 컨테이너를 추가! - [KrBlog](https://gurumee92.tistory.com/231)

![[Pasted image 20250727013117.png]]
	https://docs-archive.sc.otc.t-systems.com/usermanual/cce/cce_10_0201.html
	
- nginx 웹 서버(with helm)에 metrics 수집 설정 추가 - [Helm](https://artifacthub.io/packages/helm/bitnami/nginx)
	
```bash
# 모니터링
watch -d "kubectl get pod; echo; kubectl get servicemonitors -n monitoring"

# nginx 파드내에 컨테이너 갯수 확인 
kubectl describe pod -l app.kubernetes.io/instance=nginx

# 파라미터 파일 생성 : 서비스 모니터 방식으로 nginx 모니터링 대상을 등록하고, export 는 9113 포트 사용
# The chart can deploy ServiceMonitor objects for integration with Prometheus Operator installations. To do so, set the value metrics.serviceMonitor.enabled=true. 
cat <<EOT > nginx-values.yaml
metrics:
  enabled: true

  service:
	port: 9113

  serviceMonitor:
	enabled: true
	namespace: monitoring
	interval: 10s
EOT

# 배포
helm upgrade nginx bitnami/nginx --reuse-values -f nginx-values.yaml

# 확인
kubectl get pod,svc,ep
kubectl get servicemonitor -n monitoring nginx
kubectl get servicemonitor -n monitoring nginx -o json | jq
kubectl get servicemonitor -n monitoring nginx -o yaml | kubectl neat

# 
kubectl krew install view-secret
kubectl get secret  -n monitoring
kubectl view-secret -n monitoring prometheus-kube-prometheus-stack-prometheus
kubectl view-secret -n monitoring prometheus-kube-prometheus-stack-prometheus | zcat | more
kubectl view-secret -n monitoring prometheus-kube-prometheus-stack-prometheus | zcat | grep nginx -A 20

# [운영서버 EC2] 메트릭 확인 >> 프로메테우스에서 Target 확인
## nginx sub_status url 접속해보기
NGINXIP=$(kubectl get pod -l app.kubernetes.io/instance=nginx -o jsonpath="{.items[0].status.podIP}")
curl -s http://$NGINXIP:9113/metrics # nginx_connections_active Y 값 확인해보기
curl -s http://$NGINXIP:9113/metrics | grep ^nginx_connections_active

# nginx 파드내에 컨테이너 갯수 확인 : metrics 컨테이너 확인
kubectl get pod -l app.kubernetes.io/instance=nginx
kubectl describe pod -l app.kubernetes.io/instance=nginx

# 접속 주소 확인 및 접속
echo -e "Nginx WebServer URL = https://nginx.$MyDomain"
curl -s https://nginx.$MyDomain
kubectl stern deploy/nginx

# 반복 접속
while true; do curl -s https://nginx.$MyDomain -I | head -n 1; date; sleep 1; done
```
	
- 서비스 모니터링 생성 후 3분 정도 후에 프로메테우스 웹서버에서 State → Targets 에 nginx 서비스 모니터 추가 확인
![[Pasted image 20250727013147.png]]

- State → Configuration : nginx 검색 후 job 확인
	- (참고) job_name : serviceMonitor/monitoring/nginx/0
		
```bash
- job_name: serviceMonitor/monitoring/nginx/0
  honor_timestamps: true
  track_timestamps_staleness: false
  scrape_interval: 10s
  scrape_timeout: 10s
  scrape_protocols:
  - OpenMetricsText1.0.0
  - OpenMetricsText0.0.1
  - PrometheusText1.0.0
  - PrometheusText0.0.4
  metrics_path: /metrics
  scheme: http
  enable_compression: true
  follow_redirects: true
  enable_http2: true
  relabel_configs:
  - source_labels: [job]
	separator: ;
	target_label: __tmp_prometheus_job_name
	replacement: $1
	action: replace
  - source_labels: [__meta_kubernetes_service_label_app_kubernetes_io_instance, __meta_kubernetes_service_labelpresent_app_kubernetes_io_instance]
	separator: ;
	regex: (nginx);true
	replacement: $1
	action: keep
  - source_labels: [__meta_kubernetes_service_label_app_kubernetes_io_name, __meta_kubernetes_service_labelpresent_app_kubernetes_io_name]
	separator: ;
	regex: (nginx);true
	replacement: $1
	action: keep
  - source_labels: [__meta_kubernetes_endpoint_port_name]
	separator: ;
	regex: metrics
	replacement: $1
	action: keep
  - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
	separator: ;
	regex: Node;(.*)
	target_label: node
	replacement: ${1}
	action: replace
  - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
	separator: ;
	regex: Pod;(.*)
	target_label: pod
	replacement: ${1}
	action: replace
  - source_labels: [__meta_kubernetes_namespace]
	separator: ;
	target_label: namespace
	replacement: $1
	action: replace
  - source_labels: [__meta_kubernetes_service_name]
	separator: ;
	target_label: service
	replacement: $1
	action: replace
  - source_labels: [__meta_kubernetes_pod_name]
	separator: ;
	target_label: pod
	replacement: $1
	action: replace
  - source_labels: [__meta_kubernetes_pod_container_name]
	separator: ;
	target_label: container
	replacement: $1
	action: replace
  - source_labels: [__meta_kubernetes_pod_phase]
	separator: ;
	regex: (Failed|Succeeded)
	replacement: $1
	action: drop
  - source_labels: [__meta_kubernetes_service_name]
	separator: ;
	target_label: job
	replacement: ${1}
	action: replace
  - separator: ;
	target_label: endpoint
	replacement: metrics
	action: replace
  - source_labels: [__address__, __tmp_hash]
	separator: ;
	regex: (.+);
	target_label: __tmp_hash
	replacement: $1
	action: replace
  - source_labels: [__tmp_hash]
	separator: ;
	modulus: 1
	target_label: __tmp_hash
	replacement: $1
	action: hashmod
  - source_labels: [__tmp_hash]
	separator: ;
	regex: "0"
	replacement: $1
	action: keep
  kubernetes_sd_configs:
  - role: endpoints
	kubeconfig_file: ""
	follow_redirects: true
	enable_http2: true
	namespaces:
	  own_namespace: false
	  names:
	  - default
```
	
- 설정이 자동으로 반영되는 원리 : 주요 config 적용 필요 시 reloader 동작!

```bash
#
kubectl describe pod -n monitoring prometheus-kube-prometheus-stack-prometheus-0
...
  config-reloader:
	Container ID:  containerd://55ef5f8170f20afd38c01f136d3e5674115b8593ce4c0c30c2f7557e702ee852
	Image:         quay.io/prometheus-operator/prometheus-config-reloader:v0.72.0
	Image ID:      quay.io/prometheus-operator/prometheus-config-reloader@sha256:89a6c7d3fd614ee1ed556f515f5ecf2dba50eec9af418ac8cd129d5fcd2f5c18
	Port:          8080/TCP
	Host Port:     0/TCP
	Command:
	  /bin/prometheus-config-reloader
	Args:
	  --listen-address=:8080
	  --reload-url=http://127.0.0.1:9090/-/reload
	  --config-file=/etc/prometheus/config/prometheus.yaml.gz
	  --config-envsubst-file=/etc/prometheus/config_out/prometheus.env.yaml
	  --watched-dir=/etc/prometheus/rules/prometheus-kube-prometheus-stack-prometheus-rulefiles-0
...
```
	
- 쿼리 : 애플리케이션, Graph → nginx_ 입력 시 다양한 메트릭 추가 확인 : nginx_connections_active 등
	
```bash
# nginx scale out : Targets 확인
kubectl scale deployment nginx --replicas 2

# 쿼리 Table -> Graph
nginx_up
sum(nginx_up)

nginx_http_requests_total
nginx_connections_active
```
	

- 프로메테우스 PromQL - [Docs](https://prometheus.io/docs/prometheus/latest/querying/basics/) , [Blog](https://devthomas.tistory.com/15)
- 프로메테우스 메트릭 종류 (4종) : Counter, Gauge, Histogram, Summary  - [Link](https://prometheus.io/docs/concepts/metric_types/) [Blog](https://gurumee92.tistory.com/241)
	- 게이지 Gauge : 특정 시점의 값을 표현하기 위해서 사용하는 메트릭 타입, CPU 온도나 메모리 사용량에 대한 현재 시점 값
	- 카운터 Counter : 누적된 값을 표현하기 위해 사용하는 메트릭 타입, 증가 시 구간 별로 변화(추세) 확인, 계속 증가 → 함수 등으로 활용
	- 서머리 Summary : 구간 내에 있는 메트릭 값의 빈도, 중앙값 등 통계적 메트릭
	- 히스토그램 Histogram : 사전에 미리 정의한 구간 내에 있는 메트릭 값의 빈도를 측정 → 함수로 측정 포맷을 변경
- PromQL Query - [Docs](https://prometheus.io/docs/prometheus/latest/querying/basics/) [Operator](https://prometheus.io/docs/prometheus/latest/querying/operators/) [Example](https://prometheus.io/docs/prometheus/latest/querying/examples/)
	- Label Matchers : = , ! = , =~ 정규표현식
	
```bash
# 예시
node_memory_Active_bytes
node_memory_Active_bytes{instance="192.168.1.188:9100"}
node_memory_Active_bytes{instance!="192.168.1.188:9100"}

# 정규표현식
node_memory_Active_bytes{instance=~"192.168.+"}
node_memory_Active_bytes{instance=~"192.168.1.+"}

# 다수 대상
node_memory_Active_bytes{instance=~"192.168.1.188:9100|192.168.2.170:9100"}
node_memory_Active_bytes{instance!~"192.168.1.188:9100|192.168.2.170:9100"}

# 여러 조건 AND
kube_deployment_status_replicas_available{namespace="kube-system"}
kube_deployment_status_replicas_available{namespace="kube-system", deployment="coredns"}
```
	
- Binary Operators 이진 연산자 - [Link](https://prometheus.io/docs/prometheus/latest/querying/operators/#binary-operators)
	- 산술 이진 연산자 : + - * / * ^
	- 비교 이진 연산자 : = =  ! = > < > = < =
	- 논리/집합 이진 연산자 : and 교집합 , or 합집합 , unless 차집합

```bash
# 산술 이진 연산자 : + - * / * ^
node_memory_Active_bytes
node_memory_Active_bytes/1024
node_memory_Active_bytes/1024/1024

# 비교 이진 연산자 : = =  ! = > < > = < =
nginx_http_requests_total
nginx_http_requests_total > 100
nginx_http_requests_total > 10000

# 논리/집합 이진 연산자 : and 교집합 , or 합집합 , unless 차집합
kube_pod_status_ready
kube_pod_container_resource_requests

kube_pod_status_ready == 1
kube_pod_container_resource_requests > 1

kube_pod_status_ready == 1 or kube_pod_container_resource_requests > 1
kube_pod_status_ready == 1 and kube_pod_container_resource_requests > 1
```
	
- Aggregation Operators 집계 연산자 - [Link](https://prometheus.io/docs/prometheus/latest/querying/operators/#aggregation-operators)
	- `sum` (calculate sum over dimensions) : 조회된 값들을 모두 더함
	- `min` (select minimum over dimensions) : 조회된 값에서 가장 작은 값을 선택
	- `max` (select maximum over dimensions) : 조회된 값에서 가장 큰 값을 선택
	- `avg` (calculate the average over dimensions) : 조회된 값들의 평균 값을 계산
	- `group` (all values in the resulting vector are 1) : 조회된 값을 모두 ‘1’로 바꿔서 출력
	- `stddev` (calculate population standard deviation over dimensions) : 조회된 값들의 모 표준 편차를 계산
	- `stdvar` (calculate population standard variance over dimensions) : 조회된 값들의 모 표준 분산을 계산
	- `count` (count number of elements in the vector) : 조회된 값들의 갯수를 출력 / 인스턴스 벡터에서만 사용 가능
	- `count_values` (count number of elements with the same value) : 같은 값을 가지는 요소의 갯수를 출력
	- `bottomk` (smallest k elements by sample value) : 조회된 값들 중에 가장 작은 값들 k 개 출력
	- `topk` (largest k elements by sample value) : 조회된 값들 중에 가장 큰 값들 k 개 출력
	- `quantile` (calculate φ-quantile (0 ≤ φ ≤ 1) over dimensions) : 조회된 값들을 사분위로 나눠서 (0 < $ < 1)로 구성하고, $에 해당 하는 요소들을 출력
	
```bash
#
node_memory_Active_bytes

# 출력 값 중 Top 3
topk(3, node_memory_Active_bytes)

# 출력 값 중 하위 3
bottomk(3, node_memory_Active_bytes)
bottomk(3, node_memory_Active_bytes>0)

# node 그룹별: by
node_cpu_seconds_total
node_cpu_seconds_total{mode="user"}
node_cpu_seconds_total{mode="system"}

avg(node_cpu_seconds_total)
avg(node_cpu_seconds_total) by (instance)
avg(node_cpu_seconds_total{mode="user"}) by (instance)
avg(node_cpu_seconds_total{mode="system"}) by (instance)

#
nginx_http_requests_total
sum(nginx_http_requests_total)
sum(nginx_http_requests_total) by (instance)

# 특정 내용 제외하고 출력 : without
nginx_http_requests_total
sum(nginx_http_requests_total) without (instance)
sum(nginx_http_requests_total) without (instance,container,endpoint,job,namespace)
```
	
- Time series selectors : Instant/Range vector selectors, Time Durations, Offset modifier, @ modifier - [Link](https://prometheus.io/docs/prometheus/latest/querying/basics/#time-series-selectors)
	- 인스턴스 벡터 Instant Vector : 시점에 대한 메트릭 값만을 가지는 데이터 타입
	- 레인지 벡터 Range Vector : 시간의 구간을 가지는 데이터 타입
	- 시간 단위 : ms, s, m(주로 분 사용), h, d, w, y
	
```bash
# 시점 데이터
node_cpu_seconds_total

# 15초 마다 수집하니 아래는 지난 4회차/8회차의 값 출력
node_cpu_seconds_total[1m]
node_cpu_seconds_total[2m]
```
	
- 활용
	
	```bash
	# 서비스 정보 >> 네임스페이스별 >> cluster_ip 별
	kube_service_info
	count(kube_service_info)
	count(kube_service_info) by (namespace)
	count(kube_service_info) by (cluster_ip)
	
	# 컨테이너가 사용 메모리 -> 파드별
	container_memory_working_set_bytes
	sum(container_memory_working_set_bytes)
	sum(container_memory_working_set_bytes) by (pod)
	topk(5,sum(container_memory_working_set_bytes) by (pod))
	topk(5,sum(container_memory_working_set_bytes) by (pod))/1024/1024
	```
![[Pasted image 20250727013336.png]]
sum(container_memory_working_set_bytes) by (pod)


