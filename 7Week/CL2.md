# ClusterLoader2 (CL2) : Kubernetes 부하 테스트 도구이자 공식 K8s 확장성 및 성능 테스트 프레임워크 - [Link](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2)
- 테스트는 클러스터의 원하는 상태 집합(예: 10,000개의 포드, 2,000개의 클러스터 IP 서비스 또는 5개의 데몬 세트를 실행)을 정의하고 각 상태에 도달하는 속도(포드 처리량)를 지정합니다.
- CL2는 또한 측정할 성능 특성을 정의합니다( 자세한 내용은 [측정 목록](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2#Measurement) 참조 ).
- 마지막으로, CL2는 [Prometheus를](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2#prometheus-metrics) 사용한 테스트 중에 클러스터에 대한 추가적인 관측성을 제공합니다.
- **Flags**
    - **Required**
        - `kubeconfig` - path to the kubeconfig file.
        - `testconfig` - path to the test config file. This flag can be used multiple times if more than one test should be run.
        - `provider` - Cluster provider. Options are: gce, gke, kind, kubemark, aws, local, vsphere, skeleton
    - **Optional**
        - `nodes` - number of nodes in the cluster. If not provided, test will assign the number of schedulable cluster nodes.
        - `report-dir` - path to a directory where summary files should be stored. If not specified, summaries are printed to the standard log.
        - `mastername` - Name of the master node.
        - `masterip` - DNS Name or IP of the master node.
        - `testoverrides` - path to file with overrides.
        - `kubelet-port` - TCP port of the kubelet to use (_default: 10250_).
- Tests
    - **Test definition**
    - **Modules**
    - **Object template**
    - **Overrides**
        - `export CL2_ACCESS_TOKENS_QPS=5`
- 사용 가능한 측정값
    - **APIAvailabilityMeasurement**
        
        이 측정값은 클러스터 제어 평면의 가용성에 대한 정보를 수집합니다.
        
        이를 측정하는 두 가지 방법이 약간 다릅니다.
        
        - 클러스터 수준 가용성, 여기서 우리는 주기적으로 API 호출을 발행합니다 `/readyz`
        - 호스트 수준 가용성은 각 제어 평면의 호스트 `/readyz` 엔드포인트를 주기적으로 폴링하는 것입니다.
            - 이렇게 하려면 [exec 서비스를](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2/pkg/execservice) 활성화해야 합니다.
    - **APIResponsivenessPrometheusSimple**
        
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
        
    - **Timer**
        
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

# 준비 - [Go](https://go.dev/dl/) , [Install](https://go.dev/doc/install)
## myk8s-control-plane 컨테이너 내부에 go 설치 후 진행 실패
```bash
#
docker exec -it myk8s-control-plane bash
----------------------------------------
apt update && apt install -y tree vim git wget

# 
wget https://go.dev/dl/go1.24.1.linux-arm64.tar.gz
tar -C /usr/local -xzf go1.24.1.linux-arm64.tar.gz
export PATH=$PATH:/usr/local/go/bin

#
go version
go version go1.23.7 linux/arm64

#
git clone https://github.com/kubernetes/perf-tests.git
cd perf-tests/
tree -L 2
cd clusterloader2

# 많은 보안 강화된 Ubuntu/Cloud 환경에서 /tmp 파티션이 noexec로 마운트됩니다.
# noexec인 경우, /tmp에 저장된 바이너리는 실행 권한이 있어도 실행이 차단돼요.
mount | grep /tmp # 출력에 noexec 옵션 확인
mount -o remount,exec /tmp # noexec 해제
mount | grep /tmp

#
go run cmd/clusterloader.go -h

----------------------------------------
```

## (옵션) hostPC에 직접 설정
```bash
# go 설치
go version                                        
go version go1.24.3 darwin/arm64

#
git clone https://github.com/kubernetes/perf-tests.git
cd perf-tests/
tree -L 2
cd clusterloader2

```

## (옵션) CL2 Getting Started : 기본 환경 설치, GVM - [Link](https://github.com/kubernetes/perf-tests/blob/master/clusterloader2/docs/GETTING_STARTED.md)

### In this tutorial, we will:
    - Set-up perf-tests repository for local development
    - Create **single node** cluster using [Kind](https://kind.sigs.k8s.io/)
    - Implement a **simple CL2 test** and **run** it
    - Run load test on **100 nodes cluster**
### **Clone perf-tests repository**
```bash
#
git clone https://github.com/kubernetes/perf-tests.git
cd perf-tests
tree -L 2
tree -L 1
```
### Install GVM
```bash
# 1. 필수 의존성 설치
brew install curl git mercurial make binutils bison gcc

# 2. GVM 설치
bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)

# 3. 셸 설정파일에 추가 (zsh 기준)
echo '[[ -s "$HOME/.gvm/scripts/gvm" ]] && source "$HOME/.gvm/scripts/gvm"' >> ~/.zshrc
source ~/.zshrc

# 설치 확인
gvm version
Go Version Manager v1.0.22 installed at /Users/gasida/.gvm

```

## Install golang with specific version (1.24.3 was tested in this tutorial):
```bash
# GVM 초기화 및 설치 준비
## gvm은 내부적으로 Go 부트스트랩용 초기 버전를 먼저 설치하고, 이를 이용해 최신 Go를 빌드합니다.
gvm install go1.21.0 -B
gvm use go1.21.0

# 바로 1.24.3 설치 못하며, 중간 버전 필요
gvm install go1.22.6
gvm use go1.22.6

# 목표 버전 설치
gvm install go1.24.3 -B
gvm use go1.24.3

#
gvm list
gvm use go1.24.3 --default

#
cd kind/perf-tests                                 
pwd                
/Users/gasida/Downloads/vagrant/kind/perf-tests

# Next, add perf-tests repository to GOPATH:
## GVM은 내부 GOPATH (~/.gvm/pkgsets/.../global/src) 아래에 다음처럼 링크 생성
## 이제 Go에서 다음처럼 임포트할 수 있음 import "k8s.io/perf-tests/clusterloader2"
gvm linkthis k8s.io/perf-tests
/Users/gasida/.gvm/pkgsets/go1.24.3/global/src/k8s.io/perf-tests -> /Users/gasida/Downloads/vagrant/kind/perf-tests

```


# Prepare simple test to run - [Link](https://github.com/kubernetes/perf-tests/blob/master/clusterloader2/docs/GETTING_STARTED.md#prepare-simple-test-to-run)
- Let's prepare our first test config (**config.yaml**). This test will:
    - Create **one namespace**
    - Create a **single deployment** with **10 pods** inside that namespace
    - Measure **startup latency** of these **pod**
- config.yaml 파일 작성 : pod 생성 및 기동 시간(latency) 측정
    - **측정 시작** (Pod startup 및 pod 상태 추적)
    - **Deployment 생성** (10개 파드 생성, 1 QPS 속도)
    - **Pod들이 Running 상태가 될 때까지 기다림**
    - **Pod startup latency 최종 측정**
```bash
cat << EOF > config.yaml
name: test

namespace:
  number: 1 # 네임스페이스 1개

tuningSets: # 부하(tuning)를 조절하는 방식 정의
- name: Uniform1qps
  qpsLoad:
    qps: 1 # 초당 1개의 오브젝트를 생성

steps:
- name: Start measurements  # Step 1 : 측정 시작
  measurements: # 두 개의 메트릭을 수집 준비
  - Identifier: PodStartupLatency # 레이블 group=test-pod 을 가진 pod의 기동 시간 측정
    Method: PodStartupLatency
    Params:
      action: start
      labelSelector: group = test-pod
      threshold: 20s
  - Identifier: WaitForControlledPodsRunning # Deployment가 관리하는 pod들이 Running 상태가 될 때까지 대기하는 측정 시작
    Method: WaitForControlledPodsRunning
    Params:
      action: start
      apiVersion: apps/v1
      kind: Deployment
      labelSelector: group = test-deployment
      operationTimeout: 120s
- name: Create deployment  # Step 2: 디플로이먼트 생성 , 실제로 테스트 리소스를 생성하는 단계
  phases:
  - namespaceRange: # 하나의 namespace에서 test-deployment라는 이름으로 deployment를 생성
      min: 1
      max: 1
    replicasPerNamespace: 1
    tuningSet: Uniform1qps # 초당 1개의 리소스를 생성 (Uniform1qps 사용)
    objectBundle:
    - basename: test-deployment
      objectTemplatePath: "deployment.yaml"
      templateFillMap:
        Replicas: 10 # deployment.yaml 템플릿 파일을 바탕으로 Replicas=10으로 파드를 생성
- name: Wait for pods to be running  # Step 3: Pod 상태 대기
  measurements:
  - Identifier: WaitForControlledPodsRunning
    Method: WaitForControlledPodsRunning # deployment에 의해 생성된 pod들이 Running 상태가 되는 것을 기다림
    Params:
      action: gather  # 앞서 시작한 WaitForControlledPodsRunning 측정값을 수집(gather) 하는 단계
- name: Measure pod startup latency  # Step 4: Pod 시작 시간 측정
  measurements:
  - Identifier: PodStartupLatency # PodStartupLatency 메트릭을 수집(gather) 하여 측정 종료
    Method: PodStartupLatency # Pod 생성 요청 시점부터 Running 상태가 될 때까지의 소요 시간을 측정
    Params:
      action: gather
EOF
```

- deployment.yaml
    - `{{.Name}}`, `{{.Replicas}}` 등은 Go 템플릿 문법으로, `config.yaml`에서 `templateFillMap`에 의해 동적으로 채워짐.
```bash
cat << EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{.Name}} # 이 deployment의 이름. 템플릿으로 되어 있어 실행 시 test-deployment 등이 동적으로 삽입
  labels:
    group: test-deployment # group: test-deployment 라는 라벨은, 테스트 시 이 리소스를 식별하기 위해 사용
spec:
  replicas: {{.Replicas}} # config.yaml의 templateFillMap.Replicas: 10에 따라 10으로 대체됨
  selector:
    matchLabels:
      group: test-pod
  template:
    metadata:
      labels:
        group: test-pod
    spec:
      containers:
      - image: registry.k8s.io/pause:3.9
        name: {{.Name}}
EOF
```

# Execute test :  go run cmd/clusterloader.go -h  [Link](https://github.com/kubernetes/perf-tests/blob/master/clusterloader2/docs/GETTING_STARTED.md#execute-test)
```bash

# 모니터링
watch -d kubectl get ns,pod -A

#
go run cmd/clusterloader.go -h
go run cmd/clusterloader.go --testconfig=config.yaml --provider=kind --kubeconfig=/etc/kubernetes/admin.conf --v=2
I0824 07:53:24.675272   11219 simple_test_executor.go:411] Resources cleanup time: 15.017290506s
I0824 07:53:24.675584   11219 simple_test_executor.go:414] Tearing down dependencies
I0824 07:53:24.675658   11219 simple_test_executor.go:418] All dependencies torn down successfully, cleanup time: 74.667µs
I0824 07:53:24.676512   11219 clusterloader.go:253] --------------------------------------------------------------------------------
I0824 07:53:24.676568   11219 clusterloader.go:254] Test Finished
I0824 07:53:24.676578   11219 clusterloader.go:255]   Test: config.yaml
I0824 07:53:24.676588   11219 clusterloader.go:256]   Status: Success
I0824 07:53:24.676647   11219 clusterloader.go:260] --------------------------------------------------------------------------------

JUnit report was created: /perf-tests/clusterloader2/junit.xml
I0824 07:53:24.682590   11219 prometheus.go:331] Get snapshot from Prometheus
I0824 07:53:24.682725   11219 exec_service.go:130] Exec service: tearing down service

# At the end of clusterloader output you should see pod startup latency:
I0824 07:55:15.061694   14010 simple_test_executor.go:97] PodStartupLatency_PodStartupLatency: {
  "version": "1.0",
  "dataItems": [
    {
      "data": {
        "Perc50": 1000,
        "Perc90": 1000,
        "Perc99": 1000
      },
      "unit": "ms",
      "labels": {
        "Metric": "schedule_to_run"
      }
    },
    {
      "data": {
        "Perc50": 484.376386,
        "Perc90": 1497.292761,
        "Perc99": 1514.466886
      },
      "unit": "ms",
      "labels": {
        "Metric": "run_to_watch"
      }
    },
    {
      "data": {
        "Perc50": 1484.376386,
        "Perc90": 1508.610344,
        "Perc99": 1514.466886
      },
      "unit": "ms",
      "labels": {
        "Metric": "schedule_to_watch"
      }
    },
    {
      "data": {
        "Perc50": 1484.376386,
        "Perc90": 1508.610344,
        "Perc99": 1514.466886
      },
      "unit": "ms",
      "labels": {
        "Metric": "pod_startup"
      }
    },
    {
      "data": {
        "Perc50": 0,
        "Perc90": 0,
        "Perc99": 0
      },
      "unit": "ms",
      "labels": {
        "Metric": "create_to_schedule"
      }
    }
  ]
}
```

## Execute test 실행 시 출력 내용 : exec 파드 3개 기동 → 디플로이먼트에 파드 10개 생성 → 디플로이먼트 삭제 → exec 파드 삭제
```bash 
go run cmd/clusterloader.go --testconfig=config.yaml --provider=kind --kubeconfig=${HOME}/.kube/config --v=2

I0804 06:35:04.671542    8007 dns_performance-k8s-hostnames.go:58] Registering measurement: DNS Performance for K8s Hostnames
I0804 06:35:04.672109    8007 network_performance_measurement.go:90] Registering Network Performance Measurement
I0804 06:35:04.674121    8007 clusterloader.go:272] Listening on 8000
I0804 06:35:04.678870    8007 envvar.go:172] "Feature gate default state" feature="ClientsAllowCBOR" enabled=false
I0804 06:35:04.678908    8007 envvar.go:172] "Feature gate default state" feature="ClientsPreferCBOR" enabled=false
I0804 06:35:04.678917    8007 envvar.go:172] "Feature gate default state" feature="InformerResourceVersion" enabled=false
I0804 06:35:04.678920    8007 envvar.go:172] "Feature gate default state" feature="WatchListClient" enabled=false
I0804 06:35:04.696728    8007 clusterloader.go:168] ClusterConfig.Nodes set to 1
I0804 06:35:04.698003    8007 clusterloader.go:174] ClusterConfig.MasterName set to myk8s-control-plane
E0804 06:35:04.699403    8007 clusterloader.go:185] Getting master external ip error: didn't find any ExternalIP master IPs
I0804 06:35:04.700797    8007 clusterloader.go:192] ClusterConfig.MasterInternalIP set to [172.18.0.2]
I0804 06:35:04.700808    8007 clusterloader.go:300] Using config: {ClusterConfig:{KubeConfigPath:/Users/gasida/.kube/config RunFromCluster:false Nodes:1 Provider:0x1400000e590 EtcdCertificatePath:/etc/srv/kubernetes/pki/etcd-apiserver-server.crt EtcdKeyPath:/etc/srv/kubernetes/pki/etcd-apiserver-server.key EtcdInsecurePort:2382 MasterIPs:[] MasterInternalIPs:[172.18.0.2] MasterName:myk8s-control-plane DeleteStaleNamespaces:false DeleteAutomanagedNamespaces:true APIServerPprofByClientEnabled:true KubeletPort:10250 K8SClientsNumber:1 SkipClusterVerification:false} ReportDir: ExecServiceConfig:{Enable:true ImageRegistry:registry.k8s.io} ModifierConfig:{OverwriteTestConfig:[] SkipSteps:[]} PrometheusConfig:{TearDownServer:true EnableServer:false EnablePushgateway:false ScrapeEtcd:false ScrapeNodeExporter:false ScrapeWindowsNodeExporter:false ScrapeKubelets:false ScrapeMasterKubelets:false ScrapeKubeProxy:true KubeProxySelectorKey:component ScrapeKubeStateMetrics:false ScrapeMetricsServerMetrics:false ScrapeNodeLocalDNS:false ScrapeAnet:false ScrapeNetworkPolicies:false ScrapeMastersWithPublicIPs:false APIServerScrapePort:443 SnapshotProject: AdditionalMonitorsPath: StorageClassProvisioner:kubernetes.io/gce-pd StorageClassVolumeType:pd-ssd PVCStorageClass:ssd ReadyTimeout:15m0s PrometheusMemoryRequest:10Gi} OverridePaths:[]}
I0804 06:35:04.703355    8007 cluster.go:75] Listing cluster nodes:
I0804 06:35:04.703379    8007 cluster.go:87] Name: myk8s-control-plane, internalIP: 172.18.0.2, externalIP: , isSchedulable: true
I0804 06:35:04.705662    8007 framework.go:74] Creating framework with 1 clients and "/Users/gasida/.kube/config" kubeconfig.
I0804 06:35:04.708779    8007 exec_service.go:86] Exec service: setting up service!
I0804 06:35:04.717152    8007 framework.go:276] Applying templates for "manifest/exec_deployment.yaml"
I0804 06:35:04.717171    8007 framework.go:287] Applying manifest/exec_deployment.yaml
I0804 06:35:04.729047    8007 reflector.go:376] Caches populated for *v1.Pod from *v1.PodStore: namespace(cluster-loader), labelSelector(feature = exec)
I0804 06:35:04.777756    8007 wait_for_pods.go:64] Exec service: namespace(cluster-loader), labelSelector(feature = exec): starting with timeout: 1m59.947733833s
I0804 06:35:14.780258    8007 wait_for_pods.go:122] Exec service: namespace(cluster-loader), labelSelector(feature = exec): Pods: 3 out of 3 created, 3 running (3 updated), 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady 
I0804 06:35:14.780322    8007 exec_service.go:122] Exec service: service set up successfully!
W0804 06:35:14.780996    8007 imagepreload.go:92] No images specified. Skipping image preloading
I0804 06:35:14.786875    8007 clusterloader.go:454] Test config successfully dumped to: generatedConfig_test.yaml
I0804 06:35:14.786887    8007 clusterloader.go:243] --------------------------------------------------------------------------------
I0804 06:35:14.786890    8007 clusterloader.go:244] Running config.yaml
I0804 06:35:14.786892    8007 clusterloader.go:245] --------------------------------------------------------------------------------
I0804 06:35:14.787143    8007 simple_test_executor.go:58] AutomanagedNamespacePrefix: test-rj3xx0
I0804 06:35:14.794678    8007 simple_test_executor.go:162] Step "[step: 01] Start measurements" started
I0804 06:35:14.800663    8007 wait_for_controlled_pods.go:257] WaitForControlledPodsRunning: starting wait for controlled pods measurement...
I0804 06:35:14.800909    8007 pod_startup_latency.go:148] PodStartupLatency: labelSelector(group = test-pod): starting pod startup latency measurement...
I0804 06:35:14.802276    8007 shared_informer.go:313] Waiting for caches to sync for PodsIndexer
I0804 06:35:14.802929    8007 reflector.go:376] Caches populated for <unspecified> from pkg/mod/k8s.io/client-go@v0.32.3/tools/cache/reflector.go:251
I0804 06:35:14.806257    8007 reflector.go:376] Caches populated for *v1.Pod from pkg/mod/k8s.io/client-go@v0.32.3/tools/cache/reflector.go:251
I0804 06:35:14.806582    8007 reflector.go:376] Caches populated for *v1.ReplicaSet from pkg/mod/k8s.io/client-go@v0.32.3/tools/cache/reflector.go:251
I0804 06:35:14.902475    8007 shared_informer.go:320] Caches are synced for PodsIndexer
I0804 06:35:14.904458    8007 reflector.go:376] Caches populated for apps/v1, Resource=deployments from pkg/mod/k8s.io/client-go@v0.32.3/tools/cache/reflector.go:251
I0804 06:35:15.003407    8007 simple_test_executor.go:183] Step "[step: 01] Start measurements" ended
I0804 06:35:15.005672    8007 simple_test_executor.go:162] Step "[step: 02] Create deployment" started
I0804 06:35:15.013476    8007 wait_for_pods.go:64] WaitForControlledPodsRunning: namespace(test-rj3xx0-1), controlledBy(test-deployment-0): starting with timeout: 1m59.999992833s
I0804 06:35:16.008448    8007 simple_test_executor.go:183] Step "[step: 02] Create deployment" ended
I0804 06:35:16.008501    8007 simple_test_executor.go:162] Step "[step: 03] Wait for pods to be running" started
I0804 06:35:16.012855    8007 wait_for_controlled_pods.go:288] WaitForControlledPodsRunning: waiting for controlled pods measurement...
I0804 06:35:20.014729    8007 wait_for_pods.go:122] WaitForControlledPodsRunning: namespace(test-rj3xx0-1), controlledBy(test-deployment-0): Pods: 10 out of 10 created, 3 running (3 updated), 7 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady 
I0804 06:35:25.018757    8007 wait_for_pods.go:122] WaitForControlledPodsRunning: namespace(test-rj3xx0-1), controlledBy(test-deployment-0): Pods: 10 out of 10 created, 10 running (10 updated), 0 pending scheduled, 0 not scheduled, 0 inactive, 0 terminating, 0 unknown, 0 runningButNotReady 
I0804 06:35:25.018808    8007 wait_for_controlled_pods.go:365] WaitForControlledPodsRunning: running 1, deleted 0, timeout: 0, failed: 0
I0804 06:35:25.018821    8007 wait_for_controlled_pods.go:370] WaitForControlledPodsRunning: maxDuration=10.005361542s, operationTimeout=2m0s, ratio=0.08
I0804 06:35:25.018840    8007 wait_for_controlled_pods.go:384] WaitForControlledPodsRunning: 1/1 Deployments are running with all pods
I0804 06:35:25.018852    8007 simple_test_executor.go:183] Step "[step: 03] Wait for pods to be running" ended
I0804 06:35:25.018867    8007 simple_test_executor.go:162] Step "[step: 04] Measure pod startup latency" started
I0804 06:35:25.020381    8007 pod_startup_latency.go:242] PodStartupLatency: labelSelector(group = test-pod): gathering pod startup latency measurement...
I0804 06:35:25.028842    8007 phase_latency.go:141] PodStartupLatency: 10 worst pod_startup latencies: [{test-rj3xx0-1/test-deployment-0-5b677c889d-p7rs2 3.667268s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pdzjb 4.671876s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rl69v 4.680727s} {test-rj3xx0-1/test-deployment-0-5b677c889d-4k98l 5.691513s} {test-rj3xx0-1/test-deployment-0-5b677c889d-ht9ss 6.688202s} {test-rj3xx0-1/test-deployment-0-5b677c889d-5zvxg 7.705742s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rslpv 7.718895s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pbmk2 8.718412s} {test-rj3xx0-1/test-deployment-0-5b677c889d-nm7db 9.715026s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rsrdr 9.718689s}]
I0804 06:35:25.028869    8007 phase_latency.go:146] PodStartupLatency: perc50: 6.688202s, perc90: 9.715026s, perc99: 9.718689s; threshold 20s
I0804 06:35:25.028962    8007 phase_latency.go:141] PodStartupLatency: 10 worst create_to_schedule latencies: [{test-rj3xx0-1/test-deployment-0-5b677c889d-p7rs2 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rl69v 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-4k98l 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-ht9ss 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-5zvxg 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pdzjb 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rslpv 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pbmk2 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-nm7db 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rsrdr 0s}]
I0804 06:35:25.028970    8007 phase_latency.go:146] PodStartupLatency: perc50: 0s, perc90: 0s, perc99: 0s
I0804 06:35:25.028974    8007 phase_latency.go:141] PodStartupLatency: 10 worst schedule_to_run latencies: [{test-rj3xx0-1/test-deployment-0-5b677c889d-p7rs2 2s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rl69v 3s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pdzjb 4s} {test-rj3xx0-1/test-deployment-0-5b677c889d-4k98l 5s} {test-rj3xx0-1/test-deployment-0-5b677c889d-ht9ss 5s} {test-rj3xx0-1/test-deployment-0-5b677c889d-5zvxg 6s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rslpv 7s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pbmk2 8s} {test-rj3xx0-1/test-deployment-0-5b677c889d-nm7db 8s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rsrdr 9s}]
I0804 06:35:25.029016    8007 phase_latency.go:146] PodStartupLatency: perc50: 5s, perc90: 8s, perc99: 9s
I0804 06:35:25.029021    8007 phase_latency.go:141] PodStartupLatency: 10 worst run_to_watch latencies: [{test-rj3xx0-1/test-deployment-0-5b677c889d-pdzjb 671.876ms} {test-rj3xx0-1/test-deployment-0-5b677c889d-4k98l 691.513ms} {test-rj3xx0-1/test-deployment-0-5b677c889d-pbmk2 718.412ms} {test-rj3xx0-1/test-deployment-0-5b677c889d-rsrdr 718.689ms} {test-rj3xx0-1/test-deployment-0-5b677c889d-rslpv 718.895ms} {test-rj3xx0-1/test-deployment-0-5b677c889d-p7rs2 1.667268s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rl69v 1.680727s} {test-rj3xx0-1/test-deployment-0-5b677c889d-ht9ss 1.688202s} {test-rj3xx0-1/test-deployment-0-5b677c889d-5zvxg 1.705742s} {test-rj3xx0-1/test-deployment-0-5b677c889d-nm7db 1.715026s}]
I0804 06:35:25.029072    8007 phase_latency.go:146] PodStartupLatency: perc50: 718.895ms, perc90: 1.705742s, perc99: 1.715026s
I0804 06:35:25.029077    8007 phase_latency.go:141] PodStartupLatency: 10 worst schedule_to_watch latencies: [{test-rj3xx0-1/test-deployment-0-5b677c889d-p7rs2 3.667268s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pdzjb 4.671876s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rl69v 4.680727s} {test-rj3xx0-1/test-deployment-0-5b677c889d-4k98l 5.691513s} {test-rj3xx0-1/test-deployment-0-5b677c889d-ht9ss 6.688202s} {test-rj3xx0-1/test-deployment-0-5b677c889d-5zvxg 7.705742s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rslpv 7.718895s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pbmk2 8.718412s} {test-rj3xx0-1/test-deployment-0-5b677c889d-nm7db 9.715026s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rsrdr 9.718689s}]
I0804 06:35:25.029104    8007 phase_latency.go:146] PodStartupLatency: perc50: 6.688202s, perc90: 9.715026s, perc99: 9.718689s
I0804 06:35:25.029358    8007 phase_latency.go:141] PodStartupLatency: 10 worst pod_startup latencies: [{test-rj3xx0-1/test-deployment-0-5b677c889d-p7rs2 3.667268s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pdzjb 4.671876s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rl69v 4.680727s} {test-rj3xx0-1/test-deployment-0-5b677c889d-4k98l 5.691513s} {test-rj3xx0-1/test-deployment-0-5b677c889d-ht9ss 6.688202s} {test-rj3xx0-1/test-deployment-0-5b677c889d-5zvxg 7.705742s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rslpv 7.718895s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pbmk2 8.718412s} {test-rj3xx0-1/test-deployment-0-5b677c889d-nm7db 9.715026s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rsrdr 9.718689s}]
I0804 06:35:25.029370    8007 phase_latency.go:146] PodStartupLatency: perc50: 6.688202s, perc90: 9.715026s, perc99: 9.718689s; threshold 20s
I0804 06:35:25.029375    8007 phase_latency.go:141] PodStartupLatency: 10 worst create_to_schedule latencies: [{test-rj3xx0-1/test-deployment-0-5b677c889d-nm7db 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rsrdr 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-p7rs2 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rl69v 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-4k98l 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-ht9ss 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-5zvxg 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pdzjb 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rslpv 0s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pbmk2 0s}]
I0804 06:35:25.029420    8007 phase_latency.go:146] PodStartupLatency: perc50: 0s, perc90: 0s, perc99: 0s
I0804 06:35:25.029426    8007 phase_latency.go:141] PodStartupLatency: 10 worst schedule_to_run latencies: [{test-rj3xx0-1/test-deployment-0-5b677c889d-p7rs2 2s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rl69v 3s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pdzjb 4s} {test-rj3xx0-1/test-deployment-0-5b677c889d-4k98l 5s} {test-rj3xx0-1/test-deployment-0-5b677c889d-ht9ss 5s} {test-rj3xx0-1/test-deployment-0-5b677c889d-5zvxg 6s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rslpv 7s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pbmk2 8s} {test-rj3xx0-1/test-deployment-0-5b677c889d-nm7db 8s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rsrdr 9s}]
I0804 06:35:25.029485    8007 phase_latency.go:146] PodStartupLatency: perc50: 5s, perc90: 8s, perc99: 9s
I0804 06:35:25.029490    8007 phase_latency.go:141] PodStartupLatency: 10 worst run_to_watch latencies: [{test-rj3xx0-1/test-deployment-0-5b677c889d-pdzjb 671.876ms} {test-rj3xx0-1/test-deployment-0-5b677c889d-4k98l 691.513ms} {test-rj3xx0-1/test-deployment-0-5b677c889d-pbmk2 718.412ms} {test-rj3xx0-1/test-deployment-0-5b677c889d-rsrdr 718.689ms} {test-rj3xx0-1/test-deployment-0-5b677c889d-rslpv 718.895ms} {test-rj3xx0-1/test-deployment-0-5b677c889d-p7rs2 1.667268s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rl69v 1.680727s} {test-rj3xx0-1/test-deployment-0-5b677c889d-ht9ss 1.688202s} {test-rj3xx0-1/test-deployment-0-5b677c889d-5zvxg 1.705742s} {test-rj3xx0-1/test-deployment-0-5b677c889d-nm7db 1.715026s}]
I0804 06:35:25.029496    8007 phase_latency.go:146] PodStartupLatency: perc50: 718.895ms, perc90: 1.705742s, perc99: 1.715026s
I0804 06:35:25.029500    8007 phase_latency.go:141] PodStartupLatency: 10 worst schedule_to_watch latencies: [{test-rj3xx0-1/test-deployment-0-5b677c889d-p7rs2 3.667268s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pdzjb 4.671876s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rl69v 4.680727s} {test-rj3xx0-1/test-deployment-0-5b677c889d-4k98l 5.691513s} {test-rj3xx0-1/test-deployment-0-5b677c889d-ht9ss 6.688202s} {test-rj3xx0-1/test-deployment-0-5b677c889d-5zvxg 7.705742s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rslpv 7.718895s} {test-rj3xx0-1/test-deployment-0-5b677c889d-pbmk2 8.718412s} {test-rj3xx0-1/test-deployment-0-5b677c889d-nm7db 9.715026s} {test-rj3xx0-1/test-deployment-0-5b677c889d-rsrdr 9.718689s}]
I0804 06:35:25.029512    8007 phase_latency.go:146] PodStartupLatency: perc50: 6.688202s, perc90: 9.715026s, perc99: 9.718689s
I0804 06:35:25.029529    8007 phase_latency.go:141] PodStartupLatency: 0 worst create_to_schedule latencies: []
I0804 06:35:25.029532    8007 phase_latency.go:146] PodStartupLatency: perc50: 0s, perc90: 0s, perc99: 0s
I0804 06:35:25.029535    8007 phase_latency.go:141] PodStartupLatency: 0 worst schedule_to_run latencies: []
I0804 06:35:25.029549    8007 phase_latency.go:146] PodStartupLatency: perc50: 0s, perc90: 0s, perc99: 0s
I0804 06:35:25.029556    8007 phase_latency.go:141] PodStartupLatency: 0 worst run_to_watch latencies: []
I0804 06:35:25.029560    8007 phase_latency.go:146] PodStartupLatency: perc50: 0s, perc90: 0s, perc99: 0s
I0804 06:35:25.029563    8007 phase_latency.go:141] PodStartupLatency: 0 worst schedule_to_watch latencies: []
I0804 06:35:25.029565    8007 phase_latency.go:146] PodStartupLatency: perc50: 0s, perc90: 0s, perc99: 0s
I0804 06:35:25.029569    8007 phase_latency.go:141] PodStartupLatency: 0 worst pod_startup latencies: []
I0804 06:35:25.029571    8007 phase_latency.go:146] PodStartupLatency: perc50: 0s, perc90: 0s, perc99: 0s; threshold 20s
I0804 06:35:25.029598    8007 simple_test_executor.go:183] Step "[step: 04] Measure pod startup latency" ended
I0804 06:35:25.029608    8007 simple_test_executor.go:97] PodStartupLatency_PodStartupLatency: {
  "version": "1.0",
  "dataItems": [
    {
      "data": {
        "Perc50": 6688.202,
        "Perc90": 9715.026,
        "Perc99": 9718.689
      },
      "unit": "ms",
      "labels": {
        "Metric": "pod_startup"
      }
    },
    {
      "data": {
        "Perc50": 0,
        "Perc90": 0,
        "Perc99": 0
      },
      "unit": "ms",
      "labels": {
        "Metric": "create_to_schedule"
      }
    },
    {
      "data": {
        "Perc50": 5000,
        "Perc90": 8000,
        "Perc99": 9000
      },
      "unit": "ms",
      "labels": {
        "Metric": "schedule_to_run"
      }
    },
    {
      "data": {
        "Perc50": 718.895,
        "Perc90": 1705.742,
        "Perc99": 1715.026
      },
      "unit": "ms",
      "labels": {
        "Metric": "run_to_watch"
      }
    },
    {
      "data": {
        "Perc50": 6688.202,
        "Perc90": 9715.026,
        "Perc99": 9718.689
      },
      "unit": "ms",
      "labels": {
        "Metric": "schedule_to_watch"
      }
    }
  ]
}
I0804 06:35:25.029649    8007 simple_test_executor.go:97] StatelessPodStartupLatency_PodStartupLatency: {
  "version": "1.0",
  "dataItems": [
    {
      "data": {
        "Perc50": 6688.202,
        "Perc90": 9715.026,
        "Perc99": 9718.689
      },
      "unit": "ms",
      "labels": {
        "Metric": "pod_startup"
      }
    },
    {
      "data": {
        "Perc50": 0,
        "Perc90": 0,
        "Perc99": 0
      },
      "unit": "ms",
      "labels": {
        "Metric": "create_to_schedule"
      }
    },
    {
      "data": {
        "Perc50": 5000,
        "Perc90": 8000,
        "Perc99": 9000
      },
      "unit": "ms",
      "labels": {
        "Metric": "schedule_to_run"
      }
    },
    {
      "data": {
        "Perc50": 718.895,
        "Perc90": 1705.742,
        "Perc99": 1715.026
      },
      "unit": "ms",
      "labels": {
        "Metric": "run_to_watch"
      }
    },
    {
      "data": {
        "Perc50": 6688.202,
        "Perc90": 9715.026,
        "Perc99": 9718.689
      },
      "unit": "ms",
      "labels": {
        "Metric": "schedule_to_watch"
      }
    }
  ]
}
I0804 06:35:25.029666    8007 simple_test_executor.go:97] StatefulPodStartupLatency_PodStartupLatency: {
  "version": "1.0",
  "dataItems": [
    {
      "data": {
        "Perc50": 0,
        "Perc90": 0,
        "Perc99": 0
      },
      "unit": "ms",
      "labels": {
        "Metric": "schedule_to_watch"
      }
    },
    {
      "data": {
        "Perc50": 0,
        "Perc90": 0,
        "Perc99": 0
      },
      "unit": "ms",
      "labels": {
        "Metric": "pod_startup"
      }
    },
    {
      "data": {
        "Perc50": 0,
        "Perc90": 0,
        "Perc99": 0
      },
      "unit": "ms",
      "labels": {
        "Metric": "create_to_schedule"
      }
    },
    {
      "data": {
        "Perc50": 0,
        "Perc90": 0,
        "Perc99": 0
      },
      "unit": "ms",
      "labels": {
        "Metric": "schedule_to_run"
      }
    },
    {
      "data": {
        "Perc50": 0,
        "Perc90": 0,
        "Perc99": 0
      },
      "unit": "ms",
      "labels": {
        "Metric": "run_to_watch"
      }
    }
  ]
}
I0804 06:35:40.044779    8007 simple_test_executor.go:411] Resources cleanup time: 15.015121875s
I0804 06:35:40.044851    8007 simple_test_executor.go:414] Tearing down dependencies
I0804 06:35:40.044864    8007 simple_test_executor.go:418] All dependencies torn down successfully, cleanup time: 13.25µs
I0804 06:35:40.044877    8007 clusterloader.go:253] --------------------------------------------------------------------------------
I0804 06:35:40.044884    8007 clusterloader.go:254] Test Finished
I0804 06:35:40.044888    8007 clusterloader.go:255]   Test: config.yaml
I0804 06:35:40.044892    8007 clusterloader.go:256]   Status: Success
I0804 06:35:40.044897    8007 clusterloader.go:260] --------------------------------------------------------------------------------

JUnit report was created: /Users/gasida/Downloads/vagrant/kind/perf-tests/clusterloader2/junit.xml
I0804 06:35:40.046967    8007 prometheus.go:331] Get snapshot from Prometheus
I0804 06:35:40.046987    8007 exec_service.go:130] Exec service: tearing down service
```

```bash
#  전체 흐름 요약
초기화

커스텀 메트릭 측정 도구 등록 (dns_performance, network_performance)

클러스터 정보 추출 (master internal IP, 노드 수)

Exec 서비스 배포

manifest/exec_deployment.yaml 배포 후 3개 Pod 정상 기동

테스트 시작

config.yaml 테스트 정의에 따라 순차 실행

[Step 01] 메트릭 측정 시작

PodStartupLatency, WaitForControlledPodsRunning 등 설정

[Step 02] Deployment 생성

Deployment 1개, 총 10개 Pod 생성됨

[Step 03] Pod 기동 완료까지 대기

Pod 상태 모니터링 (10개 모두 Running)

[Step 04] Pod 기동 시간 측정

pod_startup_latency 등 여러 지표 계산

테스트 종료 및 리소스 정리

```

```bash 
# 성능 측정 주요 지표 의미
"labels": {
  "Metric": "pod_startup"
}
"Perc50": 6688.202,
"Perc90": 9715.026,
"Perc99": 9718.689

# Metric	의미
pod_startup	Pod 생성 시점부터 Ready 되기까지 전체 시간
create_to_schedule	Pod이 생성되고 스케줄러가 배정될 때까지
schedule_to_run	스케줄 배정 → 컨테이너 실행 시작까지
run_to_watch	컨테이너 실행 → 워처(watch)가 인지할 때까지
schedule_to_watch	스케줄 배정 → 워처 인지까지 (End-to-End)

예: pod_startup 지표에서 Perc90 = 9715.026ms 는 전체 Pod의 90%가 9.7초 이내에 Ready 상태가 되었음을 의미합니다.
```

```bash 
# 주의할 점
StatefulPodStartupLatency가 0인 이유는 해당 테스트에 statefulset 생성이 없기 때문입니다.

create_to_schedule = 0s는 kind 환경에서 scheduler가 매우 빠르게 동작함을 보여줍니다.

퍼포먼스는 로컬(KIND, M1 등) 환경보다 VM/클라우드 환경에서 더 의미 있습니다.
```

# JUnit report was created: ./perf-tests/clusterloader2/junit.xml

```bash 
<?xml version="1.0" encoding="UTF-8"?>
  <testsuite name="ClusterLoaderV2" tests="0" failures="0" errors="0" time="25.263">
      <testcase name="test overall (config.yaml)" classname="ClusterLoaderV2" time="25.258129916"></testcase>
      <testcase name="test: [step: 01] Start measurements [00] - PodStartupLatency" classname="ClusterLoaderV2" time="0.106681792"></testcase>
      <testcase name="test: [step: 01] Start measurements [01] - WaitForControlledPodsRunning" classname="ClusterLoaderV2" time="0.208683"></testcase>
      <testcase name="test: [step: 02] Create deployment" classname="ClusterLoaderV2" time="1.002760167"></testcase>
      <testcase name="test: [step: 03] Wait for pods to be running [00] - WaitForControlledPodsRunning" classname="ClusterLoaderV2" time="9.010379459"></testcase>
      <testcase name="test: [step: 04] Measure pod startup latency [00] - PodStartupLatency" classname="ClusterLoaderV2" time="0.010719208"></testcase>
  </testsuite>
```

- 상위 요소: `<testsuite>`
    - `name="ClusterLoaderV2"`: 테스트 수트 이름입니다. clusterloader2 실행을 의미합니다.
    - `tests="0"`: **명시적인 테스트 실패/성공 판별은 없었음**을 의미합니다. 대부분의 측정이 메트릭 수집 중심일 경우 이렇게 나올 수 있습니다.
    - `failures="0"`, `errors="0"`: 테스트 실패나 오류 없이 성공했음을 의미합니다.
    - `time="25.263"`: 전체 테스트 소요 시간 (초 단위)
- 개별 테스트 케이스: `<testcase>`
    - 각 테스트 단계별 실행 시간과 식별자를 포함한 항목입니다.
    - test overall.. : 전체 테스트(`config.yaml`) 수행 시간 , 약 25.26초 걸림
    - Step 01 - 측정 시작
        - `PodStartupLatency` 및 `WaitForControlledPodsRunning` 측정 시작 작업
        - 설정 및 초기화 단계로서 각각 0.1초, 0.2초 소요됨
    - Step 02 - 디플로이먼트 생성
        - `deployment.yaml` 템플릿을 바탕으로 리소스 생성
        - 약 1초 소요됨
    - Step 03 - Pod가 Running 될 때까지 대기
        - 생성된 pod들이 모두 `Running` 상태가 될 때까지 대기
        - 이 과정에서 kube-scheduler, container runtime 등이 관여
        - 9초 소요 → 클러스터 성능에 따라 주요 관찰 포인트
    - Step 04 - Pod Startup Latency 측정
        - 앞서 수집 시작한 `PodStartupLatency` 메트릭을 수집(gather)
        - 측정 결과 확인에 0.01초 소요

|단계|설명|시간|
|---|---|---|
|전체 테스트|config.yaml 기반 전체 실행|25.26s|
|측정 시작|PodStartupLatency 및 WaitForControlledPodsRunning 시작|~0.3s|
|디플로이먼트 생성|Deployment 오브젝트 생성|1s|
|Pod 상태 대기|Pod가 모두 Running 상태가 될 때까지 대기|9s|
|측정 결과 수집|Pod startup latency 수집|0.01s|


# `도전과제6` CL2에 다양한 테스트 해보기    - (참고) Cilium 버전별 성능 테스트 자동화 - ci: Allow for running scale test for up to 1k nodes - [Link](https://github.com/cilium/cilium/pull/40227) , [CL2-Test](https://github.com/cilium/cilium/actions/runs/16073350779/job/45362759880)

^66f017

