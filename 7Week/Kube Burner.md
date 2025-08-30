- 소개 : Kubernetes **performance** and scale test orchestration framework written in golang : `v1.17.3` - [Github](https://github.com/kube-burner/kube-burner) , [Home](https://kube-burner.github.io/kube-burner/v1.17.1/) , [examples](https://github.com/kube-burner/kube-burner/tree/main/examples)
    - Create, delete, read, and patch Kubernetes resources at scale.
    - Prometheus metric collection and indexing.
    - Measurements.
    - Alerting.
    - Kube-burner is a binary application written in Golang that makes extensive usage of the official k8s client library, [client-go](https://github.com/kubernetes/client-go).

# kube-burner 설치
```bash
#
git clone https://github.com/kube-burner/kube-burner.git
cd kube-burner

# 바이너리 설치(추천) : mac M1
curl -LO https://github.com/kube-burner/kube-burner/releases/download/v1.17.3/kube-burner-V1.17.3-darwin-arm64.tar.gz # mac M
tar -xvf kube-burner-V1.17.3-darwin-arm64.tar.gz

curl -LO https://github.com/kube-burner/kube-burner/releases/download/v1.17.3/kube-burner-V1.17.3-linux-x86_64.tar.gz # Windows
tar -xvf kube-burner-V1.17.3-linux-x86_64.tar.gz

sudo cp kube-burner /usr/local/bin

kube-burner -h
  check-alerts Evaluate alerts for the given time range
  completion   Generates completion scripts for bash shell
  destroy      Destroy old namespaces labeled with the given UUID.
  health-check Check for Health Status of the cluster
  help         Help about any command
  import       Import metrics tarball
  index        Index kube-burner metrics
  init         Launch benchmark
  measure      Take measurements for a given set of resources without running workload
  version      Print the version number of kube-burner

# 버전 확인 : 혹은 go run cmd/kube-burner/kube-burner.go -h
kube-burner version
Version: 1.17.3
```

# 시나리오 1 : 디플로이먼트 1개(파드 1개) 생성 → 삭제 , jobIterations qps burst 의미 확인

```bash
#
cat << EOF > s1-config.yaml
global:
  measurements:
    - name: none

jobs:
  - name: create-deployments
    jobType: create
    jobIterations: 1  # How many times to execute the job , 해당 job을 5번 반복 실행
    qps: 1            # Limit object creation queries per second , 	초당 최대 요청 수 (평균 속도 제한) - qps: 10이면 초당 10개 요청
    burst: 1          # Maximum burst for throttle , 순간적으로 처리 가능한 요청 최대치 (버퍼) - burst: 20이면 한순간에 최대 20개까지 처리 가능
    namespace: kube-burner-test
    namespaceLabels: {kube-burner-job: delete-me}
    waitWhenFinished: true # false
    verifyObjects: false
    preLoadImages: true # false
    preLoadPeriod: 30s # default 1m
    objects:
      - objectTemplate: s1-deployment.yaml
        replicas: 1
EOF

#
cat << EOF > s1-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-{{ .Iteration}}-{{.Replica}}
  labels:
    app: test-{{ .Iteration }}-{{.Replica}}
    kube-burner-job: delete-me
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-{{ .Iteration}}-{{.Replica}}
  template:
    metadata:
      labels:
        app: test-{{ .Iteration}}-{{.Replica}}
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
EOF
```

```bash

# 모니터링 : 터미널, kube-ops-view
watch -d kubectl get ns,pod -A


# 부하 발생 실행 Launch benchmark
kube-burner init -h
kube-burner init -c s1-config.yaml --log-level debug
...

#
kubectl get deploy -A -l kube-burner-job=delete-me
kubectl get pod -A -l kube-burner-job=delete-me
kubectl get ns -l kube-burner-job=delete-me

#
ls kube-burner-*.log
kube-burner-86508d5e-52dc-45de-88ab-d933f48ae0c8.log
cat kube-burner-*.log


# 삭제!
## deployment 는 s1-deployment.yaml 에 metadata.labels 에 추가한 labels 로 지정
## namespace 는 config.yaml 에 job.name 값을 labels 로 지정
cat << EOF > s1-config-delete.yaml
# global:
#   measurements:
#     - name: none

jobs:
  - name: delete-deployments-namespace
    qps: 500
    burst: 500
    namespace: kube-burner-test
    jobType: delete
    waitWhenFinished: true
    objects:
    - kind: Deployment
      labelSelector: {kube-burner-job: delete-me}
      apiVersion: apps/v1
    - kind: Namespace
      labelSelector: {kube-burner-job: delete-me}
EOF

#
kube-burner init -c s1-config-delete.yaml --log-level debug

```
- **`*kube-burner init -c s1-config.yaml** --log-level debug*`
- **`*kube-burner init -c s1-config-delete.yaml** --log-level debug*`
    - `preLoadImages: false` 변경 후 실행 → 차이점 확인 후 리소스 삭제, 설정값은 유지
    - `waitWhenFinished: false` 변경 후 실행 → 차이점 확인 후 리소스 삭제, 설정값은 유지
    - `jobIterations: 5` 변경 후 실행 → 차이점 확인 후 리소스 삭제, 설정값은 유지 _⇒ s1-deployment.yaml 파일 확인_
    - `objects.replicas: 2` 변경 후 실행 → 차이점 확인 후 리소스 삭제, 설정값은 유지 _⇒ s1-deployment.yaml 파일 확인_
    - `jobIterations: 10` 변경 후 실행 → 차이점 확인 후 리소스 삭제, 설정값은 유지 _⇒ qps: 1 의미 파악 해보기_
    - `qps: 10, burst:10` 변경 후 실행 → 차이점 확인 후 리소스 삭제, 설정값은 유지 _⇒ qps: 10 의미 파악 해보기_
- **qps 와 burst** 동작 파악해보기
    - `jobIterations: 10, qps: 1, burst:10` `objects.replicas: 1` 변경 후 실행 → 동작 확인 후 리소스 삭제
    - `jobIterations: 100, qps: 1, burst:100` `objects.replicas: 1` 변경 후 실행 → 동작 확인 후 리소스 삭제
    - `jobIterations: 10, qps: 1, burst:20` `objects.replicas: 2` 변경 후 실행 → 동작 확인 후 리소스 삭제
    - **`jobIterations: 10, qps: 1, burst:10` `objects.replicas: 2`** 변경 후 실행 → **차이점** 확인 후 리소스 삭제
    - **`jobIterations: 20, qps: 2, burst:20` `objects.replicas: 2`** 변경 후 실행 → **차이점** 확인 후 리소스 삭제

# 도전과제1 kube-burner 에 다양한 옵션/파라미터 등 사용. 예) jobIterationDela How long to wait between each job iteration - [Docs](https://kube-burner.github.io/kube-burner/latest/reference/configuration/)

^b11f1e


# 시나리오 2 : 노드 1대에 최대 파드(150개) 배포 시도 1
- **`*kube-burner init -c s1-config.yaml --log-level debug*`**
- `jobIterations: **100**, qps: 300, burst: 300` `objects.replicas: 1` 변경 후 실행 _→ 모든 파드가 배포 되는지 확인_

## 문제 원인 파악
![[Pasted image 20250824210853.png]]

```bash
#
kubectl get pod -A | grep -v '1/1     Running'
...

kubectl describe pod -n kube-burner-test-99 | grep Events: -A5
...

#
kubectl describe node
...
Capacity:
  cpu:                8
  ephemeral-storage:  1055761844Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             12236104Ki
  pods:               110
Allocatable:
  cpu:                8
  ephemeral-storage:  1055761844Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             12236104Ki
  pods:               110
```

## 해결
```bash
# maxPods 항목 없으면 기본값 110개
kubectl get cm -n kube-system kubelet-config -o yaml

#
docker exec -it myk8s-control-plane bash
----------------------------------------
cat /var/lib/kubelet/config.yaml

apt update && apt install vim -y
vim /var/lib/kubelet/config.yaml
maxPods: 150

systemctl restart kubelet
systemctl status kubelet
exit
----------------------------------------

#
kubectl describe node
...
Capacity:
  cpu:                8
  ephemeral-storage:  1055761844Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             12236104Ki
  pods:               150
Allocatable:
  cpu:                8
  ephemeral-storage:  1055761844Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             12236104Ki
  pods:               150
```

* kube-burner init -c s1-config-delete.yaml --log-level debug

## 시나리오 3 : 노드 1대에 최대 파드(300개) 배포 시도 2
- **`*kube-burner init -c s1-config.yaml --log-level debug*`**
- `jobIterations: **300**, qps: 300, burst: 300` `objects.replicas: 1` 변경 후 실행 _→ 모든 파드가 배포 되는지 확인_

## 문제 원인 파악
![[Pasted image 20250824211251.png]]

```bash
#
kubectl get pod -A | grep -v '1/1     Running'
...

# maxPods: 400 상향
docker exec -it myk8s-control-plane bash
----------------------------------------
cat /var/lib/kubelet/config.yaml

apt update && apt install vim -y
vim /var/lib/kubelet/config.yaml
maxPods: 400

systemctl restart kubelet
systemctl status kubelet
exit
----------------------------------------

# 
kubectl describe pod -n kube-burner-test-250 | grep Events: -A5
Events:
  Type     Reason                  Age                  From               Message
  ----     ------                  ----                 ----               -------
  Warning  FailedScheduling        5m15s                default-scheduler  0/1 nodes are available: 1 Too many pods. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.
  Normal   Scheduled               4m51s                default-scheduler  Successfully assigned kube-burner-test-250/deployment-250-1-775b95b6b5-954kk to myk8s-control-plane
  Warning  FailedCreatePodSandBox  4m38s                kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "f8b705fa3a0e185cb831de2d9ccd2fe3e7ced6eb9d81a4227d9433cbb4344711": plugin type="ptp" failed (add): 
  ... failed to allocate for range 0: no IP addresses available in range set: 10.244.0.1-10.244.0.254

kubectl describe pod -n kube-burner-test-299 | grep Events: -A5
...

#
kubectl describe node myk8s-control-plane | grep -i podcidr
PodCIDR:                      10.244.0.0/24
```

* kube-burner init -c s1-config-delete.yaml --log-level debug

