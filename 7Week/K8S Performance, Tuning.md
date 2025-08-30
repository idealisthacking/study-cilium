# [K8S Docs] Considerations for large clusters - [Docs](https://kubernetes.io/docs/setup/best-practices/cluster-large/)

- **K8S v1.33** 단일 클러스터에 최대 수용 규모
    - **5000대 노드 이하** No more than 5,000 nodes
    - **노드 당 110개 파드 이하** No more than 110 pods per node
    - **총 파드 수 150,000개 이하** No more than 150,000 total pods
    - **총 컨테이너 수 300,000개 이하** No more than 300,000 total containers
- **Control plane 고려사항**
    - 컨트롤 플레인 노드는 ‘failure zone’ 당 1개 or 2개를 배치하고, 노드를 수평 확장하자. _⇒ 다른 Rack 에 컨트롤 노드 배치 할 것!_
    - 컨트롤 플레인은 앞단에 로드밸런서를 배치하여 컨트롤 플레인 노드 일부가 장애가 나더라도 API 호출에 문제 없게 하자. ⇒ HW LB 고민해보자!

![[Pasted image 20250824211758.png]]

* etcd 스토리지 : 대규모 클러스터의 성능을 향상시키기 위해 이벤트 객체를 별도의 전용 etcd 인스턴스에 저장할 수 있습니다 - [Docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/)

![[Pasted image 20250824211833.png]]

# 2024: 쓰기만 했던 개발자가 궁금해서 찾아본 쿠버네티스 내부 1편 - [Link](https://tech.kakaopay.com/post/jack-k8s-internals-part-1/) , 2편* - [Link](https://tech.kakaopay.com/post/jack-k8s-internals-part-2/)
![[Pasted image 20250824211945.png]]
![[Pasted image 20250824212005.png]]
## 2023 K8S Controlplane 죽이기 👍🏻 - [Blog](https://iwanhae.tistory.com/m/2)
- **서론** : kube-apiserver, etcd
    - `kube-apiserver`는 **api 서버**이다.
        - HTTP(S) 로 `yaml`,`json`,`protobuf` 등의 형식으로 외부 클라이언트 (e.g., `kube-controller-manager`, `kubelet`, `kubectl` 등등등...) 들과 통신하며 RESTful 한 설계를 (거의) 잘 따라는 api 서버이다.
    - `etcd` 는 [KV DB](https://github.com/etcd-io/etcd/blob/v3.5.9/api/etcdserverpb/rpc.pb.go#L6425-L6448) 다.
        - gRPC로 `kube-apiserver` 와 통신하면서 RAFT를 사용해 자기들 끼리 Leader를 선출하고 Leader 가 모든 요청을 Write-Ahead Logging (WAL) 방식으로 기록하고 Follower 들에게 공유하는 방식으로 Split-brain 등의 [귀찮은 시나리오](https://github.blog/2018-10-30-oct21-post-incident-analysis/)를 처리하는 전형적인 CAP theorem에서 흔히 말하는 CP/EC system이다.
    - **장점** : 정말 안정적이다.
        - 모종의 이유로 (주로 대부분은 운영자의 K8s 지식에 대한 부재) 죽을 수는 있을지언정, 절대로 데이터가 꼬여서 수동복구를 해야 하는 상황은 본 적이 없다.
    - **단점** : 성능이 별로다.
        - 안정성을 위해서 성능을 희생시킨 것도 존재하지만, 좀 더 단순하게 그냥 효율 자체가 안 좋은 부분이 종종 보인다.
        - 물론 이걸 해결하라고 하면 구조상 이슈가 있어서 고치려면 전부 다 뜯어내고 고쳐야 하지만, 어차피 프로덕션에서도 이 정도 Scale 이슈가 발생하는 클러스터는 거의 없고 문제가 안되기 때문에 아마 영원히(?) 안 고쳐질 거 같다.
- **Background** : 파드 100 vs. 10000 응답 시간/크기
    - **Response Time**
        - `kubectl get pod -A` 명령어는 어떻게 작동할까?
        - 일단 `kube-apiserver` 의 `/api/v1/pods` 로 GET 요청이 날아갈 테고, `kube-apiserver` 는 `etcd` 로 `/registry/pods` 를 prefix로 가지는 모든 Key 들을 조회하는 `Range` 요청을 날리게 된다. 만약 클러스터 전체에 Pod 이 수백 ~ 수천 개 수준이라면, 이 요청은 보통 수십 ms 안쪽으로 처리된다.
```bash
> curl -o /dev/null -s -w 'Total: %{time_total}s\n' 127.0.0.1:8001/api/v1/pods?limit=100
Total: 0.031002s # 31ms
```
		* 근데 Pod 이 1만 개가 있으면 어떨까?
```bash
> curl -o /dev/null -s -w 'Total: %{time_total}s\n' 127.0.0.1:8001/api/v1/pods?limit=10000
Total: 2.131027s # 2,131ms (100개대비 약 70배)
```
Pod Resource 한 개당 처리속도를 x, HTTPS 커넥션 수립 인증 인가 기타 등등의 처리속도를 상수 c라고 한다면, 대충 리소스의 개수가 한 개 늘어날 때마다 0.21ms 정도의 처리속도 지연이 발생하게 된다.

- **Response Size**
    - 그럼 이때 다운받아지는 파일의 크기는 어떨까? 뭘 정의했고 어떻게 불러오냐에 따라서 달라지긴 하겠지만
```bash
> curl 127.0.0.1:8001/api/v1/pods?limit=100 > /dev/null
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  455k    0  455k    0     0  14.8M      0 --:--:-- --:--:-- --:--:-- 14.8M
# 0.45MB

> curl 127.0.0.1:8001/api/v1/pods?limit=10000 > /dev/null
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 44.4M    0 44.4M    0     0  21.2M      0 --:--:--  0:00:02 --:--:-- 21.2M
# 44.4MB
```

대충 1만 개의 Pod 에 대한 리소스를 JSON 형식으로 다운로드하는 것은 44.4MB 정도 된다.

- **Problem** : 메모리 사용량! _(테스트 기준 k8s v1.27 / etcd 3.5 버전 상황)_
    - 사실 데이터 전송량 좀 커지고 응답속도 늦어지는 건 큰 문제는 아니다. 어차피 K8s 세계에서 1~2초 정도 늦어지는 건 대부분 큰 문제가 아니다. 그게 큰 문제라면 현재 시스템에 디자인 결함이 있는 것이다. 전송 데이터 커지는 것은 돈의 힘 (온프레임이라면 더 비싼 네트워크장비, 클라우드라면 더 많은 트래픽 비용)으로 해결가능하다.
    - 여기서 문제는 `kube-apiserver` 와 `etcd` 의 메모리 사용량이다.
    - `Kubernetes 1.27` `etcd 3.5` 버전이 최신인 현재를 기준으로 **양쪽 모두**
        1. 응답을 돌려주기 위해 **메모리상에 필요한 모든 데이터를 적재**한 뒤,
        2. 이를 사용자의 요청 포맷 (`protobuf` `json` `yaml` 등등)에 맞게 **변환한 것을 돌려주**고 있다.
    - 그 결과 **매 요청**마다 etcd와 kube-apiserver로부터 다음과 같은 그래프를 확인할 수 있다.

### **etcd 경우**

- **etcd**는 요청이 올 때 (거의 대부분의 상황에서) **메모리에 적재되어 있는 내용을 복사해서 응답으로 돌려**준다.
- **요청 하나를 처리**하는데 **대략 30~60MB 정도의 메모리 공간이 필요**하며 이는 **10,000 개의 Pod Resource를** **Protobuf**로 저장했을 때 필요한 **저장공간 (35MB)과 거의 유사**하다.

```bash
# etcd 의 경우
# Resident Set Size (RSS) 값에 주목 :  현재 프로세스가 실제로 RAM에 상주하고 있는 메모리의 양, 물리적으로 메모리에 적재된 메모리 크기
> pidstat --human -r -p 7 1
Linux 5.15.0-75-generic (257102d738db)  06/25/23        _x86_64_        (256 CPU)

08:28:47      UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
# 프로세스 시작 후 안정화 상태
08:28:48        0         7      0.00      0.00   10.7G  110.0M   0.7%  etcd

# kube-apiserver 프로세스 시작
08:28:58        0         7      0.00      0.00   10.7G  110.0M   0.7%  etcd
08:28:59        0         7    130.00      0.00   10.7G  111.0M   0.7%  etcd
08:29:00        0         7   9376.00      0.00   10.8G  170.5M   1.1%  etcd
08:29:01        0         7     11.00      0.00   10.8G  170.5M   1.1%  etcd
08:29:02        0         7     14.00      0.00   10.8G  170.5M   1.1%  etcd
08:29:03        0         7    100.00      0.00   10.8G  171.2M   1.1%  etcd
08:29:04        0         7      0.00      0.00   10.8G  171.2M   1.1%  etcd

# 중간 생략
08:29:23        0         7      7.00      0.00   10.8G  171.5M   1.1%  etcd

# 첫번째 curl 127.0.0.1:8001/api/v1/pods?limit=10000 (메모리 사용량 60MB 증가)
08:29:24        0         7   6941.58      0.00   10.8G  236.7M   1.5%  etcd
08:29:25        0         7      0.00      0.00   10.8G  236.7M   1.5%  etcd
08:29:26        0         7      0.00      0.00   10.8G  236.7M   1.5%  etcd
08:29:27        0         7      0.00      0.00   10.8G  236.7M   1.5%  etcd

# 두번째 curl 127.0.0.1:8001/api/v1/pods?limit=10000 (메모리 사용량 30MB 증가)
08:29:28        0         7   1162.00      0.00   10.8G  265.0M   1.7%  etcd
08:29:29        0         7      0.00      0.00   10.8G  265.0M   1.7%  etcd
08:29:30        0         7      0.00      0.00   10.8G  265.0M   1.7%  etcd
08:29:31        0         7      0.00      0.00   10.8G  265.0M   1.7%  etcd

# 세번째 curl 127.0.0.1:8001/api/v1/pods?limit=10000 (메모리 사용량 유지)
08:29:32        0         7     28.00      0.00   10.8G  265.3M   1.7%  etcd
08:29:33        0         7      0.00      0.00   10.8G  265.3M   1.7%  etcd
08:29:34        0         7      0.00      0.00   10.8G  265.3M   1.7%  etcd
08:29:35        0         7      0.00      0.00   10.8G  265.3M   1.7%  etcd

# etcd GC 실행 (메모리 사용량 90MB 감소)
08:29:36        0         7   9306.00      0.00   10.9G  187.1M   1.2%  etcd
08:29:37        0         7      2.00      0.00   10.9G  187.1M   1.2%  etcd
```

### kube-apiserver의 경우 
- `kube-apiserver` 의 경우 상황이 etcd 만큼 단순하지 않다. 요청을 받으면
    1. etcd로부터 protobuf 포맷으로 데이터를 받은 뒤
    2. **go struct** 형태로 **변환**한 것을 (deserialization, unmarshal)
    3. **다시 json** 형태로 **변환** (serialization, marshal)
- 하는 과정이 최소한으로 들어가고 상황에 따라서 버전 변환 (e.g., `v1beta1` -> `v1`) 이나 List 형식에 맞게 타입변환을 하는 과정도 필요하다 (e.g., `Pod` -> `PodList`)
- 그 결과 `kube-apiserver` 는 매 요청당 100MB 내외의 메모리를 필요로 한다.
- 이는 앞서 다운로드한 `44.4MB` 의 json 데이터의 **대략 2.5배 정도 되는 크기**이다.
```bash
# kube-apiserver 의 경우
# Resident Set Size (RSS) 값에 주목 :  현재 프로세스가 실제로 RAM에 상주하고 있는 메모리의 양, 물리적으로 메모리에 적재된 메모리 크기
> pidstat --human -r -p 7 1
Linux 5.15.0-75-generic (257102d738db)  06/25/23        _x86_64_        (256 CPU)

08:41:43      UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
# 프로세스 시작 후 안정화 상태
08:41:44        0         7      0.00      0.00    1.4G  691.4M   4.3%  kube-apiserver
08:41:45        0         7      1.00      0.00    1.4G  691.4M   4.3%  kube-apiserver
08:41:46        0         7      0.00      0.00    1.4G  691.4M   4.3%  kube-apiserver
08:41:47        0         7      0.00      0.00    1.4G  691.4M   4.3%  kube-apiserver
08:41:48        0         7      8.00      0.00    1.4G  691.4M   4.3%  kube-apiserver

# 첫번째 curl 127.0.0.1:8001/api/v1/pods?limit=10000 (메모리 사용량 120MB 증가)
08:41:49        0         7   9304.00      0.00    1.4G  750.7M   4.7%  kube-apiserver
08:41:50        0         7    346.00      0.00    1.5G  815.9M   5.1%  kube-apiserver
08:41:51        0         7      1.00      0.00    1.5G  815.9M   5.1%  kube-apiserver
08:41:52        0         7      1.00      0.00    1.5G  815.9M   5.1%  kube-apiserver

# 두번째 curl 127.0.0.1:8001/api/v1/pods?limit=10000 (메모리 사용량 170MB 증가)
08:41:53        0         7    717.00      0.00    1.5G  845.0M   5.3%  kube-apiserver
08:41:54        0         7   1645.00      0.00    1.6G  942.7M   5.9%  kube-apiserver
08:41:55        0         7    278.00      0.00    1.6G  944.0M   5.9%  kube-apiserver
08:41:56        0         7      1.00      0.00    1.6G  944.0M   5.9%  kube-apiserver

# 세번째 curl 127.0.0.1:8001/api/v1/pods?limit=10000 (메모리 사용량 70MB 증가)
08:41:57        0         7     75.00      0.00    1.6G  944.2M   5.9%  kube-apiserver
08:41:58        0         7   1493.00      0.00    1.7G 1010.8M   6.3%  kube-apiserver
08:41:59        0         7      2.00      0.00    1.7G 1010.8M   6.3%  kube-apiserver
08:42:00        0         7      7.00      0.00    1.7G 1010.8M   6.3%  kube-apiserver
08:42:01        0         7      1.00      0.00    1.7G 1010.8M   6.3%  kube-apiserver

# 네번째 curl 127.0.0.1:8001/api/v1/pods?limit=10000 (메모리 사용량 20MB 증가)
08:42:02        0         7    526.00      0.00    1.7G    1.0G   6.4%  kube-apiserver
08:42:03        0         7   2636.00      0.00    1.8G    1.1G   7.2%  kube-apiserver
08:42:04        0         7      7.00      0.00    1.8G    1.1G   7.2%  kube-apiserver
08:42:05        0         7      0.00      0.00    1.8G    1.1G   7.2%  kube-apiserver

# 다섯번째 curl 127.0.0.1:8001/api/v1/pods?limit=10000 (메모리 사용량 100MB? 증가)
08:42:06        0         7  16652.00      0.00    2.0G    1.2G   7.8%  kube-apiserver
08:42:07        0         7   1883.00      0.00    2.0G    1.2G   7.9%  kube-apiserver
08:42:08        0         7      2.00      0.00    2.0G    1.1G   7.3%  kube-apiserver
08:42:09        0         7      0.00      0.00    2.0G    1.1G   7.3%  kube-apiserver

# 여섯번째 curl 127.0.0.1:8001/api/v1/pods?limit=10000 (메모리 사용량 감소)
08:42:10        0         7    184.00      0.00    2.0G    1.0G   6.6%  kube-apiserver
08:42:11        0         7      4.00      0.00    2.0G    1.0G   6.6%  kube-apiserver
08:42:12        0         7      0.00      0.00    2.0G    1.0G   6.6%  kube-apiserver
08:42:13        0         7      0.00      0.00    2.0G    1.0G   6.6%  kube-apiserver

# 일곱번째 curl 127.0.0.1:8001/api/v1/pods?limit=10000 (메모리 사용량 유지)
08:42:14        0         7     32.00      0.00    2.0G    1.0G   6.6%  kube-apiserver
08:42:15        0         7    353.00      0.00    2.0G    1.0G   6.6%  kube-apiserver
08:42:16        0         7      4.00      0.00    2.0G    1.0G   6.6%  kube-apiserver
08:42:17        0         7      0.00      0.00    2.0G    1.0G   6.6%  kube-apiserver
08:42:18        0         7      1.00      0.00    2.0G    1.0G   6.6%  kube-apiserver

# 여덟번째 curl 127.0.0.1:8001/api/v1/pods?limit=10000 (메모리 사용량 유지)
08:42:19        0         7   1137.00      0.00    2.0G    1.1G   6.9%  kube-apiserver
08:42:20        0         7      0.00      0.00    2.0G    1.1G   6.9%  kube-apiserver
08:42:21        0         7      0.00      0.00    2.0G    1.1G   6.9%  kube-apiserver
08:42:22        0         7      0.00      0.00    2.0G    1.1G   6.9%  kube-apiserver
```

- **그래서 이게 왜 문제일까?** : 동시 요청 시 OOM Killed..
    - 물론 요청당 수십MB 씩 메모리를 필요로 하는 것은 꽤 크긴 하지만, 이는 단발성이다.
    - 요청이 끝나면 메모리는 해제되며 다음 GC 때 이 메모리는 운영체제에게 반환된다. 하지만 이런 **요청이 100개가 동시**에 들어온다면 어떨까?

### kube-apiserver의 경우

* 기껏해야 1GB 정도 사용할까 말까 했던 kube-apiserver 메모리 용량이 30초도 안 되는 시간에 그 6배인 6GB까지 가더니 cgroup으로 설정한 hard limit을 넘어가 OOM Killed 당했다.

```bash
# kube-apiserver 의 경우
# Resident Set Size (RSS) 값에 주목
> pidstat --human -r -p 7 1
Linux 5.15.0-75-generic (36ebf0245e96)  06/26/23        _x86_64_        (256 CPU)

01:52:22      UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
01:52:29        0         7      0.00      0.00    1.4G  693.6M   4.3%  kube-apiserver
01:52:30        0         7      1.00      0.00    1.4G  693.6M   4.3%  kube-apiserver
01:52:31        0         7      2.00      0.00    1.4G  693.6M   4.3%  kube-apiserver
01:52:32        0         7      5.00      0.00    1.4G  693.6M   4.3%  kube-apiserver
01:52:33        0         7      0.00      0.00    1.4G  693.6M   4.3%  kube-apiserver

# ab -c 100 -n 100 127.0.0.1:8001/api/v1/pods
01:52:34        0         7    118.00      0.00    1.4G  693.9M   4.3%  kube-apiserver
01:52:35        0         7   3114.00      0.00    1.5G  780.8M   4.9%  kube-apiserver
01:52:36        0         7   1686.00      0.00    1.5G  787.0M   4.9%  kube-apiserver
01:52:37        0         7     17.00      0.00    1.5G  787.3M   4.9%  kube-apiserver
01:52:38        0         7     23.00      0.00    1.5G  787.3M   4.9%  kube-apiserver
01:52:39        0         7     22.00      0.00    1.5G  787.3M   4.9%  kube-apiserver
01:52:40        0         7     35.00      0.00    1.5G  787.5M   4.9%  kube-apiserver
01:52:41        0         7   6526.00      0.00    1.6G  860.2M   5.4%  kube-apiserver
01:52:42        0         7  48340.00      1.00    2.1G    1.1G   7.0%  kube-apiserver
01:52:43        0         7  34056.00      0.00    2.7G    1.4G   8.7%  kube-apiserver
01:52:44        0         7  34789.00      0.00    3.9G    1.7G  10.9%  kube-apiserver
01:52:45        0         7  36829.00      0.00    4.8G    2.2G  14.2%  kube-apiserver
01:52:46        0         7  70896.00      0.00    5.2G    2.7G  17.5%  kube-apiserver
01:52:47        0         7  64348.00      1.00    5.5G    3.2G  20.4%  kube-apiserver
01:52:48        0         7  99643.00      0.00    6.1G    3.9G  24.8%  kube-apiserver
01:52:49        0         7   7634.00      1.00    6.1G    4.0G  25.8%  kube-apiserver
01:52:50        0         7   8189.11      0.00    6.4G    4.4G  28.0%  kube-apiserver
01:52:51        0         7  26415.00      0.00    6.7G    4.7G  30.3%  kube-apiserver
01:52:52        0         7  31627.45      0.00    6.8G    5.0G  32.2%  kube-apiserver
01:52:53        0         7  47195.00      0.00    7.1G    5.4G  34.5%  kube-apiserver
01:52:54        0         7  38151.00      0.00    7.4G    5.7G  36.5%  kube-apiserver
01:52:55        0         7  36775.00      0.00    7.6G    6.0G  38.2%  kube-apiserver

# OOM Killed
01:52:56        0         7   5954.00      0.00    0.0k    0.0k   0.0%  kube-apiserver
```

### etcd의 경우

* etcd의 경우도 상황이 비슷해서 같은 시간 동안 평소에 200MB 도 제대로 사용 안 하던 메모리 사용량이 그 30배인 6GB까지 치솟더니 요청을 걸어주는 클라이언트가 죽으니깐 그 상태를 유지하게 된다.

```bash
# etcd 의 경우
# Resident Set Size (RSS) 값에 주목
> pidstat --human -r -p 7 1
Linux 5.15.0-75-generic (36ebf0245e96)  06/26/23        _x86_64_        (256 CPU)

01:52:24      UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
01:52:30        0         7      0.00      0.00   10.8G  165.4M   1.0%  etcd
01:52:31        0         7      0.00      0.00   10.8G  165.4M   1.0%  etcd
01:52:32        0         7      2.00      0.00   10.8G  165.4M   1.0%  etcd
01:52:33        0         7      0.00      0.00   10.8G  165.4M   1.0%  etcd

# ab -c 100 -n 100 127.0.0.1:8001/api/v1/pods
01:52:34        0         7   1066.00      0.00   10.8G  164.3M   1.0%  etcd
01:52:35        0         7      0.00      0.00   10.8G  164.3M   1.0%  etcd
01:52:36        0         7 147810.00      0.00   11.4G  583.3M   3.6%  etcd
01:52:37        0         7 113682.00      0.00   11.7G    1.0G   6.4%  etcd
01:52:38        0         7 140902.00      0.00   12.3G    1.5G   9.8%  etcd
01:52:39        0         7 140856.00      0.00   12.7G    2.1G  13.3%  etcd
01:52:40        0         7 136235.00      0.00   13.3G    2.6G  16.6%  etcd
01:52:41        0         7  99380.20      0.00   13.7G    3.1G  19.8%  etcd
01:52:42        0         7  77303.00      0.00   14.5G    3.9G  24.9%  etcd
01:52:43        0         7  45276.00      0.00   15.1G    4.5G  28.7%  etcd
01:52:44        0         7  28677.00      0.00   16.1G    5.4G  34.6%  etcd
01:52:45        0         7  14475.00      0.00   16.6G    6.0G  38.1%  etcd
01:52:46        0         7     17.00      0.00   16.6G    6.0G  38.1%  etcd
01:52:47        0         7      1.00      0.00   16.6G    6.0G  38.1%  etcd
01:52:48        0         7      0.00      0.00   16.6G    6.0G  38.1%  etcd
01:52:49        0         7      5.00      0.00   16.6G    6.0G  38.1%  etcd
01:52:50        0         7      0.00      0.00   16.6G    6.0G  38.1%  etcd
01:52:51        0         7      1.00      0.00   16.6G    6.0G  38.1%  etcd
01:52:52        0         7      0.99      0.00   16.6G    6.0G  38.1%  etcd
01:52:53        0         7      5.00      0.00   16.6G    6.0G  38.1%  etcd
01:52:54        0         7      7.00      0.00   16.6G    6.0G  38.1%  etcd
01:52:55        0         7     21.00      0.00   16.6G    6.0G  38.1%  etcd

# kube-apiserver OOM Killed
01:52:56        0         7      0.00      0.00   16.6G    6.0G  38.1%  etcd
01:52:57        0         7      6.00      0.00   16.6G    6.0G  38.1%  etcd
01:52:58        0         7      0.00      0.00   16.6G    6.0G  38.1%  etcd
01:52:59        0         7      0.00      0.00   16.6G    6.0G  38.1%  etcd
01:53:00        0         7      0.00      0.00   16.6G    6.0G  38.1%  etcd
01:53:01        0         7      0.00      0.00   16.6G    6.0G  38.1%  etcd

# 한참 뒤 GC 수행 이후
01:59:58        0         7      0.00      0.00   16.6G  327.5M   2.0%  etcd
01:59:59        0         7      0.00      0.00   16.6G  327.5M   2.0%  etcd
```

- **왜 이런 문제가 발생하는 것일까? & 언제 문제가 발생할까?**
    - 이 모든 것의 첫 번째 원인은 **etcd**의 Transaction이다.
        - etcd는 읽는 도중에 데이터가 변경되는 상황에 대해 안전한 동시성 처리를 위해서 Range 요청이 처리되는 동안 [Store 가 변경되지 않도록 Lock을 걸어두고](https://github.com/etcd-io/etcd/blob/v3.5.9/server/mvcc/kvstore_txn.go#L42-L51), 이 **Lock 이 걸리는 시간** 동안 [Range 요청에 해당하는 데이터를 복제](https://github.com/etcd-io/etcd/blob/v3.5.9/server/mvcc/index.go#L147-L163)하는데 이 과정에서 메**모리 사용량이 늘어나게** 된다.
        - 이 과정을 만약 Zero Copy로 구현할 경우 Lock 걸리는 시간을 protobuf 형태의 데이터를 완성될 때까지로 늘어날 수밖에 없는데, 이는 성능에 심각한 저하를 가져온다.
    - 이러한 제약사항속에 열려있는 **etcd API**는 [Range 밖에 없으니](https://github.com/etcd-io/etcd/blob/v3.5.9/api/etcdserverpb/rpc.proto#L14-L21) 여기서 두 번째 문제가 발생한다.
        - `kube-apiserver` 입장에서도 이를 마땅히 처리할 방법이 없다. limits 가 크게 걸려있다면 **그 요청을 그대로 보내는 수밖에 없다.**
        - 하지만 리소스 1만 개를 불러오는 요청은 2초가량의 시간이 소모되는 작업이고 2초 안쪽으로 100건의 같은 요청이 들어온다면 메모리는 급증할 수밖에 없게 되는 것이다.

- 이 이슈가 **발생하는 조건**은 두 가지이다.
    1. 한 번에 조회 가능한 리소스의 개수가 매우 많다.
    2. 해당 리소스를 조회하는 요청이 매우 많다.
- 이러한 두 조건을 만족하는 대표적인 사례가 **비활성화된 `Pod` 이 많고, 노드 개수가 많은 상황**이다.
- 본인이 발견한 대표적인 사례는 `Airflow` 와 `Kubeflow` 이다. ~~(하여간 flow 붙은 놈들이 다 문제다.)~~

1. **한 번에 조회 가능한 리소스의 개수가 매우 많다.**
    - 먼저 첫 번째 조건에 대해서 이야기해 보면 이 둘은 설정을 제대로 안 해두면 단발성 작업 (e.g., CronJob)으로 생성한 Pod를 자동으로 삭제하지 않고 남겨둔다.
    - 이러한 Pod 이 수십 개 수준이면 별 문제가 안되지만 클러스터 관리자가 별 관심이 없다면 수천 개 정도 쌓이는 경우도 종종 발생한다. 이걸로 첫 번째 조건을 충족한다.
2. **해당 리소스를 조회하는 요청이 매우 많다.**
    - 두 번째 조건은 컨트롤러들의 동기화 전략과 인덱싱 부재와 관련된 문제이다.
    - 쿠버네티스에는 수많은 컨트롤러들이 존재하지만, 그중 **노드 개수에 비례하게 늘어나는 것**들을 몇 개 뽑아보면
        1. kubelet
        2. kube-proxy
        3. CNI agents (cilium, calico, flannel 기타 등등) 등이 존재한다.
    - 이들은 평소에는 **Watch API**를 통해서 상태가 변화되는 Pod 들에 대한 **정보만 받는 형태로 정말 효율적**으로 작동한다.
    - 하지만 모종의 이유로 Controlplane으로 가는 TCP 커넥션들이 분단된 상황을 생각해 보자. (이런 일은 Loadbalancer에 의해서 생각보다 자주 발생한다.)
    - 이때 각 노드에 존재하는 `kuelet` 과 같은 컨트롤러들은 **최신 상태를 동기화**하기 위해 `/api/v1/pods?fieldSelector=spec.nodeName=worker-node-10` 위와 같이 fieldSelector를 명시해서 `kube-apiserver` 에 요청을 보내게 되고 `kubelet` 은 자신이 원하는 데이터만 받게 되지만...
```bash
// etcd 로그
{
  "level": "warn",
  "ts": "2023-06-26T01:52:44.693121Z",
  "caller": "etcdserver/util.go:170",
  "msg": "apply request took too long",
  "took": "1.875451825s",
  "expected-duration": "100ms",
  "prefix": "read-only range ",
  "request": "key:\"/registry/pods/\" range_end:\"/registry/pods0\" ",
  "response": "range_response_count:20037 size:28870318"
}
```

- `etcd` 는 아니다. `kube-apiserver` 는 `spec.nodeName` 에 대한 **인덱싱을 해두고 있지 않기** 때문에, `etcd` 는 단순한 KV Storage 이기 때문에 `spec.nodeName=worker-node-10` 인 **Pod를 찾으려면 모든 리소스를 하나하나 검사**해 보는 방법밖에 없고 `kube-apiserver` 는 `etcd` 로부터 모든 정보를 불러오는 요청을 보내게 된다.
- 즉, `kube-apiserver` 는 살아남는다 해도 `etcd` 는 죽어버리는 상황이 발생한다는 것이다.

# 해결법 : API 관점, 운영 관점

**API 관점**

1. **Limit** / **Continue** (API 서버에 대한 최소한의 예의)
    - 첫 번째 해결법은 `limit` `continue` 활용이다. 한 번에 조회되는 양이 많아서 문제라면, 한 번에 조회되는 양을 줄이면 된다.
    - `kubectl`, `kubelet` 을 포함한 거의 모든 K8s 프로젝트는 이 둘을 활용해서 한 번에 최대 **500**개씩 조회를 한다.
    - 다음은 실제 kubectl 이 API 요청을 보내는 예시이다.
```bash
> root@36ebf0245e96:~# kubectl get po -v6
I0626 04:14:00.931736      77 loader.go:373] Config loaded from file:  /etc/kubernetes/admin.conf
I0626 04:14:01.008707      77 round_trippers.go:553] GET https://127.0.0.1:6443/api/v1/namespaces/default/pods?limit=500 200 OK in 66 milliseconds
I0626 04:14:01.105069      77 round_trippers.go:553] GET https://127.0.0.1:6443/api/v1/namespaces/default/pods?continue=eyJ2IjoibWV0YS5rOHMuaW8vdjEiLCJydiI6MzkzNDcsInN0YXJ0IjoidGVzdC01NzQ2ZDRjNTlmLTJuNTUyXHUwMDAwIn0&limit=500 200 OK in 47 milliseconds
I0626 04:14:01.198811      77 round_trippers.go:553] GET https://127.0.0.1:6443/api/v1/namespaces/default/pods?continue=eyJ2IjoibWV0YS5rOHMuaW8vdjEiLCJydiI6MzkzNDcsInN0YXJ0IjoidGVzdC01NzQ2ZDRjNTlmLTRiOHpzXHUwMDAwIn0&limit=500 200 OK in 44 milliseconds
I0626 04:14:01.297993      77 round_trippers.go:553] GET https://127.0.0.1:6443/api/v1/namespaces/default/pods?continue=eyJ2IjoibWV0YS5rOHMuaW8vdjEiLCJydiI6MzkzNDcsInN0YXJ0IjoidGVzdC01NzQ2ZDRjNTlmLTR6czh3XHUwMDAwIn0&limit=500
(이하생략)
```

2. **ResourceVersion / ResourceVersionMatch** (etcd 만큼은 살리겠다는 의지)
    - 두 번째 해결법은 `ResourceVersion` 과 `ResourceVersionMatch` 를 적당이 활용하는 것이다. ([참고](https://kubernetes.io/docs/reference/using-api/api-concepts/#resource-versions))
    - 이 둘은 Strong Consistency 가 필요하지 않은경우 유효한 해결법으로 **resourceVersion="0"** 으로 요청을 날릴 경우 `etcd` 에서 최신 데이터를 가지고 오지 않고 `kube-apiserver` [캐시에 저장된](https://danielmangum.com/posts/k8s-asa-watching-and-caching/) 데이터를 그냥 가져온다.
![[Pasted image 20250824214057.png]]
- 최신 정보가 수집된다는 보장은 없지만, 어차피 다른 컨트롤러가 Watch 하면서 (`kube-controller-manager` 는 언제나 모든 Pod을 바라보고 있다)
- 어쩔 수 없이 받게 된 정보가 kube-apiserver 캐시에 저장됨으로 나름 최신의 정보를 지속적으로 받게 된다.
```bash
# kube-apiserver
# ab -c 100 -n 200 127.0.0.1:8001/api/v1/pods?resourceVersion=0
04:35:01        0         7      6.00      0.00    1.4G  696.5M   4.4%  kube-apiserver
04:35:02        0         7      0.00      0.00    1.4G  696.5M   4.4%  kube-apiserver
04:35:03        0         7      1.00      0.00    1.4G  696.5M   4.4%  kube-apiserver
04:35:04        0         7    667.00      0.00    1.4G  706.9M   4.4%  kube-apiserver
04:35:05        0         7  14133.98      0.00    1.9G  767.0M   4.8%  kube-apiserver
04:35:06        0         7  53444.00      1.00    2.6G    1.3G   8.1%  kube-apiserver
04:35:07        0         7  25610.00      0.00    3.0G    1.5G   9.9%  kube-apiserver
04:35:08        0         7  53421.00      0.00    4.0G    2.3G  14.5%  kube-apiserver
04:35:09        0         7  41038.00      0.00    4.4G    2.5G  15.9%  kube-apiserver
04:35:10        0         7  55646.00      0.00    4.5G    2.7G  17.2%  kube-apiserver
04:35:11        0         7  57569.00      0.00    4.7G    3.1G  19.6%  kube-apiserver
04:35:12        0         7  71497.00      0.00    5.1G    3.5G  22.6%  kube-apiserver
04:35:13        0         7  32127.45      0.00    5.1G    3.7G  23.8%  kube-apiserver
04:35:14        0         7  26431.00      0.00    5.3G    3.9G  24.9%  kube-apiserver
04:35:15        0         7  65825.00      0.00    5.5G    4.3G  27.5%  kube-apiserver
04:35:16        0         7  64327.00      0.00    6.0G    4.7G  30.1%  kube-apiserver
04:35:17        0         7  56355.34      0.00    6.5G    5.3G  33.7%  kube-apiserver
04:35:18        0         7  40895.05      0.00    6.9G    5.7G  36.3%  kube-apiserver

# OOM KILLED
04:35:19        0         7  51130.00      0.00    0.0k    0.0k   0.0%  kube-apiserver
```

```bash
# etcd
# ab -c 100 -n 200 127.0.0.1:8001/api/v1/pods?resourceVersion=0
04:35:01        0         7      0.00      0.00   10.8G  164.2M   1.0%  etcd
04:35:02        0         7      3.00      0.00   10.8G  164.2M   1.0%  etcd
04:35:03        0         7      5.00      0.00   10.8G  164.2M   1.0%  etcd
04:35:04        0         7      1.00      0.00   10.8G  164.2M   1.0%  etcd
04:35:05        0         7      0.00      0.00   10.8G  164.2M   1.0%  etcd
04:35:06        0         7      4.00      0.00   10.8G  164.2M   1.0%  etcd
04:35:07        0         7      0.00      0.00   10.8G  164.2M   1.0%  etcd
04:35:08        0         7      0.00      0.00   10.8G  164.2M   1.0%  etcd
04:35:09        0         7      0.00      0.00   10.8G  164.2M   1.0%  etcd
04:35:10        0         7      2.00      0.00   10.8G  164.4M   1.0%  etcd
04:35:11        0         7      0.00      0.00   10.8G  164.4M   1.0%  etcd
04:35:13        0         7      0.00      0.00   10.8G  164.4M   1.0%  etcd
04:35:14        0         7      3.00      0.00   10.8G  164.4M   1.0%  etcd
04:35:15        0         7      1.98      0.00   10.8G  164.4M   1.0%  etcd
04:35:16        0         7      4.00      0.00   10.8G  164.4M   1.0%  etcd
04:35:17        0         7      4.00      0.00   10.8G  164.4M   1.0%  etcd
04:35:18        0         7      8.82      0.00   10.8G  164.4M   1.0%  etcd

# kube-apiserver OOM KILLED / etcd survived
04:35:19        0         7      0.00      0.00   10.8G  164.4M   1.0%  etcd
04:35:20        0         7    126.00      0.00   10.8G  164.9M   1.0%  etcd
```
- 물론 `kube-apiserver` 가 OOM으로 죽게 되는 것은 피할 수 없지만, 어차피 `kube-apiserver` 는 `stateless` 하고 죽어도 수초 안에 다시 회복됨으로 `etcd` 가 죽어서 쿼럼이 깨지는 것보다는 훨씬 덜 치명적이다. (쿼럼은 한번 깨지면 모든 etcd를 동시에 다 살려야지만 복구된다.)
- 이러한 내용들은 [Informer](https://pkg.go.dev/k8s.io/client-go/informers) 에 이미 반영된 내용들이며 `Infromer` pkg를 사용하는 대부분의 Kubernetes 프로젝트들은 이러한 내용들이 이미 반영되어 있다.

3. API Priority and Fairness (APF) (kube-apiserver 도 같이 살리겠다는 의지) - [Docs](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/)
    - 사실 이 해결법은 앞서 말한 List 상황에 해당되는 문제는 아니다. **APF** 기본적인 콘셉트는 **Rate Limit**이다.
    - 본래 `kube-apiserver` 에는 `max-requests-inflight` 가 걸려있어서 최대 동시처리 요청개수를 제한할 수 있지만... [기본값이 400](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)이다.
    - 동시 요청 100개만 넣어도 잘하면 메모리 사용량이 20GB 도 넘어가게 할 수 있는데 400은 큰 의미가 없을뿐더러 중요한 거 안 중요한 거 다 합쳐서 제약을 거는 바람에 별로 유용하지 못하다.
    - APF는 이러한 API 요청에 대한 Rate Limit을 좀 더 유연하게 가져가 보자는 맥락이다.
    - 이에 대한 자세한 사례는 독일산 DevOps 회사인 [palark](https://blog.palark.com/kubernetes-api-flow-control-management/)에서 잘 소개를 해줬는데, 대략적인 사용법은 다음 yaml을 보면 이해하기가 쉽다.
```bash
# Original Source: https://blog.palark.com/kubernetes-api-flow-control-management/
---
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: FlowSchema # Flowcontrol 을 적용할 API 요청에 대한 조건
metadata:
  name: cilium-pods
spec:
  distinguisherMethod:
    type: ByUser
  matchingPrecedence: 1000 # 하나의 요청이 두개이상의 FlowSchema 에 걸릴때 우선권
  priorityLevelConfiguration:
    name: cilium-pods # 적용할 RateLimit 룰
  rules:
    - resourceRules:
        - apiGroups:
            - "cilium.io"
          clusterScope: true
          namespaces:
            - "*"
          resources:
            - "*"
          verbs:
            - "list"
      subjects:
        - group:
            name: system:serviceaccounts:d8-cni-cilium
          kind: Group
---
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: PriorityLevelConfiguration
metadata:
  name: cilium-pods
spec:
  type: Limited # Exempt, Limited 중 택 1
  limited:
    nominalConcurrencyShares: 5
    # 본래는 assuredConcurrencyShares 였지만, 분산환경에서 코드짜기 복잡해져서 다음과 같이 적용
    # NominalCL(i)  = ceil( ServerCL * NCS(i) / sum_ncs )
    # sum_ncs = sum[limited priority level k] NCS(k)
    # CL = Concurrency Limit, NCS = nominalConcurrencyShares
    limitResponse:
      # 짧게 설명할 자신이 없어서 링크로 대체
      # https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/1040-priority-and-fairness
      queuing:
        handSize: 4
        queueLengthLimit: 50
        queues: 16
      type: Queue # 지금 당장 처리못하는 요청은 Queue 넣었다가 지연된 응답보냄. Reject 도 **선택**
```
* 문제가 되는 프로젝트의 요청만 격리해서 처리가 가능해서 꽤 유연성이 좋다.

**운영 관점**

- 이 모든 문제는 애초에 리소스 개수가 많지 않으면 별 문제가 안된다.
- 불필요한 리소스를 지속적으로 삭제해 주자.

# [OpenAI] K8S 2500대 노드 → 7500대 노드 운영하기

2018
[https://openai.com/index/scaling-kubernetes-to-2500-nodes/](https://openai.com/index/scaling-kubernetes-to-2500-nodes/) → 번역 [https://coffeewhale.com/scaling-node01](https://coffeewhale.com/scaling-node01)

- **etcd**
    - **500노드**를 넘기자, kubectl 사용 시 잦은 timeout 발생 → kube-apiserver 추가.. 10대까지 추가 : 근본 원인 해결 되지 않음!
    - etcd 가 ‘쓰기’ 동작에서 몇백 밀리세컨드의 튀는 현상 spiking 확인 → 이 latency 가 전체 클러스터의 응답 시간을 늦추게함.
![[Pasted image 20250824220925.png]]

- **etcd** 를 네트워크 저장소에서 → 서버에 **로컬 디스크 SSD**로 옮김. ⇒ 지연이 200us 줄고, **etcd 정상 동작!**
- 1000 노드까지 문제 없이 동작 → 1000 노드가 넘어가자 다시 etcd 높은 지연 시간 발생.
- kube-apiserver 가 etcd 로부터 500MB/s 넘는 데이터를 읽어 들이는 것을 확인.
- 몇 가지 슬로우 쿼리와 Events API를 List 하는 api 콜이 엄청나다는 것을 확인.
- 근본 원인은 Fluentd 와 Datadog 기본 설정에서 클러스터의 모드 노드에서 API 서버로 값을 질의하게 동작. (이 이슈는 수정되었음)
- 시스템들이 조금 덜 공격적이게 api 서버로 질의하도록 수정하여 api 서버가 정상 동작!
- Tip) k8s event api 를 etcd 클러스터와 분리하여 저장하는 것을 추천. event 생성 시 부하를 메인 etcd 클러스터에 주지 않아 성능 향상에 도움.

- **도커 이미지 pulls**
    - `kubelet`의 `--serialize-image-pulls` 플래그 : 17GB 게임 이미지 다운로드 시 다른 파드의 이미지 영향 문제 발생
        - true (기본값) : 노드에서 동시에 하나의 이미지 풀만 허용, 다음 이미지 풀은 이전 작업이 끝날 때까지 대기
        - **false** : 여러 이미지 풀을 병렬로 이미지 pull 수행
    - 이미지 pull 속도를 올리기 위해, 도커 root 디렉터리도 직접 로컬 디바이스로 연결된 SSD로 옮김
    - `kubelet`의 `--image-pull-progress-deadline` 플래그 : pull 중 진행(progress)이 없을 경우 실패로 간주할 시간 제한
        - 기본값 1분 → **30분**으로 늘림.

- **ARP Cache**
    - dmesh 에 `neighbor table overflow`라는 메세지가 발견 → ARP cache 공간이 부족
    - `/etc/sysctl.conf`에 있는 kernel 파라미터를 수정

```bash
net.ipv4.neigh.default.gc_thresh1 = 80000  # ARP/NDP 캐시가 80,000개 이상일 경우, 커널은 유휴 entry를 청소 (Soft GC 시작)
net.ipv4.neigh.default.gc_thresh2 = 90000  # 캐시가 90,000개를 넘으면 GC가 좀 더 적극적으로 수행됨
net.ipv4.neigh.default.gc_thresh3 = 100000 # 절대 최대치. 10만 개를 넘는 새로운 neighbor entry는 추가되지 않음 (즉시 실패함)


# 현재 설정 확인
sysctl net.ipv4.neigh.default.gc_thresh{1,2,3}

# 일시적 적용
sysctl -w net.ipv4.neigh.default.gc_thresh1=80000
sysctl -w net.ipv4.neigh.default.gc_thresh2=90000
sysctl -w net.ipv4.neigh.default.gc_thresh3=100000

# 영구 적용 (예: /etc/sysctl.conf 또는 /etc/sysctl.d/99-custom.conf)
echo "net.ipv4.neigh.default.gc_thresh1=80000" >> /etc/sysctl.conf
```

2021 [https://openai.com/index/scaling-kubernetes-to-7500-nodes/](https://openai.com/index/scaling-kubernetes-to-7500-nodes/) → 번역 [https://coffeewhale.com/scaling-node02](https://coffeewhale.com/scaling-node02)

- **API Servers**
    - 프로메테우스-스택에서 제공하는 그라파나 대시보드 : API server requests - HTTP 응답코드 429, 5XX 에러 알람 → 이상 파악 도움
    - api 서버와 etcd 둘 다 전용 서버(dedicated nodes) 에서 실행.
    - 각각 5개의 API 서버와 etcd 구성하여 부하를 분산 시키고 노드 장애를 대비
    - **api 서버는 꽤 많은 메모리** 사용. 7500노드에서 **각 API 서버**마다 약 **70GB 힙 메모리** 사용 확인.
    - 노드 변경(추가/제거) 시 endpoint 리소스 watch 시, 네트워크 bw 요구량은 평균 1GB/s 필요 → 1.17 **EndpointSlices** 이후 **부하 1/1000 이하로 감소**.


# Misadventures in Large Scale Cluster Performance - Shane Corbett, AWS & Dima Ilchenko, Lacework - [Youtube](https://www.youtube.com/watch?v=wvpWmWzOPiQ)

- 정리 : 동작 원리 이해 → 정확한 메트릭으로 분석 → 근본 원인 찾고 해결
    10,000 파드를 동시에 실행 시 문제 발생 → 순차 실행으로 해결
![[Pasted image 20250824221723.png]]

K8S 구성요소 마다 요청 처리 한계가 있음.
![[Pasted image 20250824221737.png]]
KCM 컨트롤러 동작 , 다른 컨트롤러도 동일하게 동작.
![[Pasted image 20250824221753.png]]

Concurrency(동시성, Thread 적정수), QPS/Burst
![[Pasted image 20250824221811.png]]
High QPS → API Server → High Concurrency → ETCD (Headroom 여유가 있어야함)
![[Pasted image 20250824221830.png]]

API Server(Client Agent) → ETCD 요청의 지연 측정 : 특히 LIST는 느림
![[Pasted image 20250824221846.png]]

API to ETCD Latency(P99) : LIST 제외한 나머지는 모두 1초 미만!
![[Pasted image 20250824221903.png]]

1000개 jobs 동시 생성 실행 시 병목이 발생하게 되면 큐가 늘어나고 대기하게 됨.
![[Pasted image 20250824223309.png]]

Work Queue Depth : 대기열 메트릭 확인
![[Pasted image 20250824223328.png]]

요청 지연 : 큐에 진입하고 → 처리되기 까지의 시간
![[Pasted image 20250824223351.png]]

Queue_duration 메트릭
![[Pasted image 20250824223409.png]]

Slow Requests : go 쓰레드가 일정 시간 동안 잠기는 영향으로 4개 쓰레드만 사용되는 시점
![[Pasted image 20250824223439.png]]

Long Work Duration : work_duration 메트릭
![[Pasted image 20250824223454.png]]
평균 처리 메트릭 확인
![[Pasted image 20250824223510.png]]

쿼리별 느린것 확인
![[Pasted image 20250824223524.png]]

50,000개의 파드를 조회 요청…
![[Pasted image 20250824223541.png]]

# 시나리오 4 : api-intensive - 파드를 생성(configmap, secret) 후 삭제
- This workload is meant to load kube-apiserver by creating pods mounting secrets and configmaps, and then delete them.
- You'll need to tweak QPS/Burst and jobIterations parameters according to the cluster size.

```bash
#
tree examples/workloads/api-intensive
examples/workloads/api-intensive
├── api-intensive.yml
└── templates
    ├── configmap.yaml
    ├── deployment_patch_add_label.json
    ├── deployment_patch_add_label.yaml
    ├── deployment_patch_add_pod_2.yaml
    ├── deployment.yaml
    ├── secret.yaml
    └── service.yaml

cd examples/workloads/api-intensive

#
cat << EOF > api-intensive-100.yml
jobs:
  - name: api-intensive
    jobIterations: 100
    qps: 100
    burst: 100
    namespacedIterations: true
    namespace: api-intensive
    podWait: false
    cleanup: true
    waitWhenFinished: true
    preLoadImages: false # true
    objects:
      - objectTemplate: templates/deployment.yaml
        replicas: 1
      - objectTemplate: templates/configmap.yaml
        replicas: 1
      - objectTemplate: templates/secret.yaml
        replicas: 1
      - objectTemplate: templates/service.yaml
        replicas: 1

  - name: api-intensive-patch
    jobType: patch
    jobIterations: 10
    qps: 100
    burst: 100
    objects:
      - kind: Deployment
        objectTemplate: templates/deployment_patch_add_label.json
        labelSelector: {kube-burner-job: api-intensive}
        patchType: "application/json-patch+json"
        apiVersion: apps/v1
      - kind: Deployment
        objectTemplate: templates/deployment_patch_add_pod_2.yaml
        labelSelector: {kube-burner-job: api-intensive}
        patchType: "application/apply-patch+yaml"
        apiVersion: apps/v1
      - kind: Deployment
        objectTemplate: templates/deployment_patch_add_label.yaml
        labelSelector: {kube-burner-job: api-intensive}
        patchType: "application/strategic-merge-patch+json"
        apiVersion: apps/v1

  - name: api-intensive-remove
    qps: 500
    burst: 500
    jobType: delete
    waitForDeletion: true
    objects:
      - kind: Deployment
        labelSelector: {kube-burner-job: api-intensive}
        apiVersion: apps/v1

  - name: ensure-pods-removal
    qps: 100
    burst: 100
    jobType: delete
    waitForDeletion: true
    objects:
      - kind: Pod
        labelSelector: {kube-burner-job: api-intensive}

  - name: remove-services
    qps: 100
    burst: 100
    jobType: delete
    waitForDeletion: true
    objects:
      - kind: Service
        labelSelector: {kube-burner-job: api-intensive}

  - name: remove-configmaps-secrets
    qps: 100
    burst: 100
    jobType: delete
    objects:
      - kind: ConfigMap
        labelSelector: {kube-burner-job: api-intensive}
      - kind: Secret
        labelSelector: {kube-burner-job: api-intensive}

  - name: remove-namespace
    qps: 100
    burst: 100
    jobType: delete
    waitForDeletion: true
    objects:
      - kind: Namespace
        labelSelector: {kube-burner-job: api-intensive}
EOF

# 수행
kube-burner init -c api-intensive-100.yml --log-level debug


cat << EOF > api-intensive-500.yml
jobs:
  - name: api-intensive
    jobIterations: 100
    qps: 500
    burst: 500
    namespacedIterations: true
    namespace: api-intensive
    podWait: false
    cleanup: true
    waitWhenFinished: true
    preLoadImages: false # true
    objects:
      - objectTemplate: templates/deployment.yaml
        replicas: 1
      - objectTemplate: templates/configmap.yaml
        replicas: 1
      - objectTemplate: templates/secret.yaml
        replicas: 1
      - objectTemplate: templates/service.yaml
        replicas: 1

  - name: api-intensive-patch
    jobType: patch
    jobIterations: 10
    qps: 500
    burst: 500
    objects:
      - kind: Deployment
        objectTemplate: templates/deployment_patch_add_label.json
        labelSelector: {kube-burner-job: api-intensive}
        patchType: "application/json-patch+json"
        apiVersion: apps/v1
      - kind: Deployment
        objectTemplate: templates/deployment_patch_add_pod_2.yaml
        labelSelector: {kube-burner-job: api-intensive}
        patchType: "application/apply-patch+yaml"
        apiVersion: apps/v1
      - kind: Deployment
        objectTemplate: templates/deployment_patch_add_label.yaml
        labelSelector: {kube-burner-job: api-intensive}
        patchType: "application/strategic-merge-patch+json"
        apiVersion: apps/v1

  - name: api-intensive-remove
    qps: 500
    burst: 500
    jobType: delete
    waitForDeletion: true
    objects:
      - kind: Deployment
        labelSelector: {kube-burner-job: api-intensive}
        apiVersion: apps/v1

  - name: ensure-pods-removal
    qps: 500
    burst: 500
    jobType: delete
    waitForDeletion: true
    objects:
      - kind: Pod
        labelSelector: {kube-burner-job: api-intensive}

  - name: remove-services
    qps: 500
    burst: 500
    jobType: delete
    waitForDeletion: true
    objects:
      - kind: Service
        labelSelector: {kube-burner-job: api-intensive}

  - name: remove-configmaps-secrets
    qps: 500
    burst: 500
    jobType: delete
    objects:
      - kind: ConfigMap
        labelSelector: {kube-burner-job: api-intensive}
      - kind: Secret
        labelSelector: {kube-burner-job: api-intensive}

  - name: remove-namespace
    qps: 500
    burst: 500
    jobType: delete
    waitForDeletion: true
    objects:
      - kind: Namespace
        labelSelector: {kube-burner-job: api-intensive}
EOF

# 수행
kube-burner init -c api-intensive-500.yml --log-level debug

```

# Kubernetes API Performance Metrics: Examples and Best Practices - [Blog](https://www.redhat.com/en/blog/kubernetes-api-performance-metrics-examples-and-best-practices)

Kubernetes에서 일반적인 애플리케이션 워크로드의 배포
![[Pasted image 20250824223722.png]]

- 사용자는 REST API나 명령줄을 사용하여 배포를 생성. User creates a Deployment using a REST API or the command line.
- API Server ([kube-apiserver](https://kubernetes.io/docs/reference/generated/kube-apiserver/)) writes the Deployment spec into etcd.
- DeploymentController는 새로운 Deployments를 watches하여 이벤트를 생성하고 Deployment Spec에 맞춰 조정하고 ReplicaSet 매니페스트를 생성한 다음 이를 API 서버에 게시하고 ReplicaSet 사양을 etcd에 기록합니다.
- ReplicaSet 컨트롤러는 ReplicaSet 생성 이벤트를 감시하고 새 Pod 매니페스트를 생성합니다. 매니페스트를 API 서버에 게시하고 Pod 사양을 etcd에 기록합니다.
- The Scheduler watches pod creation events and detects an unbound Pod. It schedules and updates the Pod's Node binding.
- 노드에서 실행되는 Kubelet은 새로운 Pod 스케줄링을 감지하고 컨테이너 런타임(예: cri-o)을 사용하여 이를 실행합니다.
- Kubelet은 컨테이너 런타임을 통해 Pod 상태를 검색하여 API 서버에 업데이트합니다.

## Observing the metrics of API Server

- How many requests does kube-apiserver receive from different clients? **API 서버 QPS**를 리소스별, 요청 타입별, 응답 코드별로 모니터링
    - 최근 **5분간** Kubernetes **API 서버**가 처리한 요청을 초당 단위로 계산하고, 리소스/응답 코드/요청 동작별로 집계.
    - The counter vector of **apiserver_request_total** can be used to monitor the requests to apiserver, from where they are coming, which action and whether they were successful
```bash
# 연습1 : 라벨(label) 과 값(value) 확인
apiserver_request_total
rate(apiserver_request_total{job="apiserver"}[5m])
irate(apiserver_request_total{job="apiserver"}[5m])

# 연습2
sum by(code) (irate(apiserver_request_total{job="apiserver"}[5m]))
sum by(verb) (irate(apiserver_request_total{job="apiserver"}[5m]))
sum by(resource) (irate(apiserver_request_total{job="apiserver"}[5m]))
sum by(resource, code, verb) (irate(apiserver_request_total{job="apiserver"}[5m]))

# 연습3 : resource 값 없는 것 제외
topk(3, sum by(resource) (irate(apiserver_request_total{job="apiserver"}[5m])))
topk(3, sum by(resource) (irate(apiserver_request_total{resource=~".+"}[5m])))

# 연습3 : 4xx, 5xx 응답 코드만 필터링
sum by(code) (rate(apiserver_request_total{code=~"[45].."}[1m]))


# 최종
sum by(resource, code, verb) (rate(apiserver_request_total{resource=~".+"}[5m]))
or
sum by(resource, code, verb) (irate(apiserver_request_total{resource=~".+"}[5m]))
```
![[Pasted image 20250824223841.png]]
![[Pasted image 20250824223852.png]]

- You can also get the **Read Request Success Rate** and **Write Request Success Rate**:
    - You can see that the POST and PUT requests of kube-controller-manager and kube-scheduler increased due to Deployments, ReplicaSets and Pods created.
    - This is due to creating, updating, and deleting primary resources via the standard HTTP verbs (POST, PUT, PATCH, DELETE).

```bash
# Read Request Success Rate : 읽기 요청(GET, LIST)에 대한 성공률 
sum(irate(apiserver_request_total{code=~"20.*",verb=~"GET|LIST"}[5m]))/sum(irate(apiserver_request_total{verb=~"GET|LIST"}[5m]))

# Write Request Success Rate : 쓰기 요청(GET, LIST, WATCH, CONNECT 제외)에 대한 성공률 
sum(irate(apiserver_request_total{code=~"20.*",verb!~"GET|LIST|WATCH|CONNECT"}[5m]))/sum(irate(apiserver_request_total{verb!~"GET|LIST|WATCH|CONNECT"}[5m]))
```

![[Pasted image 20250824223916.png]]

Slowest Requests
```bash
# apiserver_request_duration_seconds_bucket : Kubernetes API 서버가 처리한 요청의 지연 시간(latency) 을 초 단위로 측정
## *_bucket → Prometheus histogram 타입 메트릭: 여러 버킷(예: 0.1s, 0.5s, 1s, 5s …)에 누적 카운트를 기록
## 라벨: verb(요청 동작:ET, LIST, POST, PATCH 등), group, version, resource(Kubernetes API group/version/resource)..
apiserver_request_duration_seconds_bucket : latency 버킷별 누적 카운트, P50, P90, P99 latency 계산(histogram_quantile)
## 자매품 apiserver_request_duration_seconds_count : 전체 요청 수 , 요청량 추적, QPS 계산
## 자매품 apiserver_request_duration_seconds_sum : 전체 요청 처리 시간 합, 평균 latency = sum / count, 부하/성능 분석

# 
apiserver_request_duration_seconds_bucket
histogram_quantile(0.90, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (verb, le))

# 내림차순 정렬
sort_desc(histogram_quantile(0.90, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (verb, le)))


# le = “less than or equal” : 히스토그램(_bucket)에서 각 버킷의 상한값. 즉, 이 버킷에 포함된 값은 le 이하임을 의미.
## 예시) : 각 버킷은 누적 카운트(cumulative) 방식
apiserver_request_duration_seconds_bucket{verb="GET", resource="pods", le="0.1"} 1200
apiserver_request_duration_seconds_bucket{verb="GET", resource="pods", le="0.5"} 1500
apiserver_request_duration_seconds_bucket{verb="GET", resource="pods", le="1"} 1600
le="0.1" → 0.1초 이하 처리된 GET pods 요청 1,200건
le="0.5" → 0.5초 이하 처리된 요청 1,500건 (즉, 0.1~0.5초 사이 요청 300건 포함)
le="1" → 1초 이하 처리된 요청 1,600건
```

![[Pasted image 20250824223946.png]]
![[Pasted image 20250824223955.png]]

![[Pasted image 20250824224006.png]]

- **CPU Usage in CPU Seconds**
    - Another important metric is process_cpu_seconds_total, the total user and system CPU time spent in seconds:
    - You can use the above PromQL to get how many CPU seconds that apiserver, kube-controller-manager, etcd, and scheduler consume.
```bash
#
process_cpu_seconds_total
process_cpu_seconds_total{job=~"apiserver|kube-controller-manager|kube-scheduler|kube-etcd"}
irate(process_cpu_seconds_total{job=~"apiserver|kube-controller-manager|kube-scheduler|kube-etcd"}[1m])
```

![[Pasted image 20250824224044.png]]

![[Pasted image 20250824224103.png]]

- **Admission Controller :** Latency , QPS → skip
    - You also need to consider admission controller latency. The admission controller intercepts requests to the Kubernetes API server prior to persistence of the object, but after the request is authenticated and authorized.
    - [Admission controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) may be validating, mutating, or both. Mutating controllers may modify objects related to the requests they admit. Validating controllers may not.
    - Admission controllers limit requests to create, delete, modify objects. Admission controllers can also block custom verbs, such as a request that connects to a Pod over an API server proxy. Admission controllers do not (and cannot) block requests to read (get, watch or list) objects.
    - Admission checks that an object can be created or updated by verifying cluster global constraints and might set defaults depending on the cluster configuration.

```bash
# Admission Controller Latency [admit]
histogram_quantile(0.99, sum by(le, name, operation, rejected, type) (irate(apiserver_admission_controller_admission_duration_seconds_bucket{$job="kube-apiserver", type="admit"}[5m])))

# Admission Controller Latency [validate]
histogram_quantile(0.99, sum by(le, name, operation, rejected, type) (irate(apiserver_admission_controller_admission_duration_seconds_bucket{$job="kube-apiserver", type="validate"}[5m])))

# The QPS of the admission webhook.
sum by(name, operation, rejected, type) (irate(apiserver_admission_webhook_admission_duration_seconds_count[5m]))
```

## Observing the metrics between the API server and etcd datastore

- The etcd cluster stores the configuration data, state data, and metadata in Kubernetes.
- Several keys are created in the etcd datastore after creating the Deployment workload.
- Find all keys in the datastore using `etcdctl get --keys-only --prefix /`
- kube-apiserver는 kube-apiserver와 etcd 사이의 API 요청 지연 시간을 관찰하는 `etcd_request_duration_seconds_bucket`의 히스토그램 벡터를 노출합니다. 이 PromSQL에서 지연 시간을 얻을 수 있습니다:
```bash
#
k exec -it -n kube-system etcd-myk8s-control-plane -- etcdctl version
k exec -it -n kube-system etcd-myk8s-control-plane -- sh -c 'etcdctl get --keys-only --prefix /'

#
etcd_request_duration_seconds_bucket

#
sum by(le, operation) (increase(etcd_request_duration_seconds_bucket[1m]))
sum by(le, type) (increase(etcd_request_duration_seconds_bucket[1m]))
sum by(le, operation, type) (increase(etcd_request_duration_seconds_bucket[1m]))
```

![[Pasted image 20250824224221.png]]

![[Pasted image 20250824224234.png]]

![[Pasted image 20250824224245.png]]
- 위의 그래픽은 etcd 요청 지연 시간의 추세를 보여줍니다. 여러 배포를 병렬로 생성할 때 지연 시간이 증가합니다.
- 대부분의 요청 시간은 100ms 미만이지만, 일부 이상치 요청은 2s에 가깝습니다.

## Observing the metrics between API server and other components
- The kube-controller-manager, kube-scheduler and kubelet are core components of Kubernetes.
    - These components are clients interacting with **API** servers via **LIST**/**WATCH** mechanism.
    - The metric `rest_client_request_duration_seconds_bucket` measures the **latency** or **duration** in seconds for calls to the **API** server.
        - 위에서 알아본 k8s api 메트릭은 api 서버(파드) 입장에서의 메트릭.
        - 이번 경우에는 client 관점에서 api 서버(파드)에 요청 시의 메트릭.
    - It is a good way to monitor the communications between the kube-controller-manager, kubelet, or scheduler, and other components and the API, and check whether these requests are being responded to within the expected time.
```bash
#
rest_client_request_duration_seconds_bucket
## 자매품 : rest_client_request_duration_seconds_count , rest_client_request_duration_seconds_sum
## 자매품 : rest_client_request_size_bytes_bucket_{bucket,count,sum}

# 
rest_client_request_duration_seconds_bucket{job=~"kube-controller-manager|kube-scheduler|kubelet"}
rate(rest_client_request_duration_seconds_bucket{job=~"kube-controller-manager|kube-scheduler|kubelet"}[1m])
sum by(le, service, verb) (rate(rest_client_request_duration_seconds_bucket{job=~"kube-controller-manager|kube-scheduler|kubelet"}[1m]))
sum by(le, pod, verb) (rate(rest_client_request_duration_seconds_bucket{job=~"kube-controller-manager|kube-scheduler|kubelet"}[1m]))
sum by(le, pod, verb) (rate(rest_client_request_duration_seconds_bucket{job=~"kube-controller-manager|kube-scheduler|kubelet",pod=~".+"}[1m]))
sum by(le, pod) (rate(rest_client_request_duration_seconds_bucket{job=~"kube-controller-manager|kube-scheduler|kubelet",pod=~".+"}[1m]))
sort_desc(sum by(le, pod) (rate(rest_client_request_duration_seconds_bucket{job=~"kube-controller-manager|kube-scheduler|kubelet",pod=~".+"}[1m])))

# 최종
histogram_quantile(0.99, sum by(le, service, verb) (rate(rest_client_request_duration_seconds_bucket{job=~"kube-controller-manager|kube-scheduler|kubelet"}[1m])))
```

![[Pasted image 20250824235509.png]]

* In the above graphic, the POST and PUT durations increase a lot more than during normal conditions due to 400 Deployments created in parallel.
```bash
# You can get the QPS from a different client perspective:
sum by(method, container, code) (rate(rest_client_requests_total{job=~"kube-controller-manager|kube-scheduler|kubelet"}[1m]))
```

![[Pasted image 20250824235543.png]]

## Work queue addition rate
- How fast is the controller-manager performing these actions?
- 다음 쿼리를 사용하여 분위수를 통한 작업 **대기열 지연 시간**을 나타냅니다. 이 메트릭은 kube-controller-manager가 단위 시간당 수행하는 필요한 작업 수입니다. 비율이 높을수록 일부 노드의 클러스터에 문제가 있음을 나타냅니다.

```bash
#
workqueue_queue_duration_seconds_bucket
workqueue_queue_duration_seconds_bucket{job=~"kube-controller-manager", name=~"deployment|replicaset|endpoint|endpoint_slice|endpoint_slice_mirroring"}
rate(workqueue_queue_duration_seconds_bucket{job=~"kube-controller-manager", name=~"deployment|replicaset|endpoint|endpoint_slice|endpoint_slice_mirroring"}[5m])
sum by(name, le) (rate(workqueue_queue_duration_seconds_bucket{job=~"kube-controller-manager", name=~"deployment|replicaset|endpoint|endpoint_slice|endpoint_slice_mirroring"}[5m]))

#
histogram_quantile(0.99, sum by(name, le) (rate(workqueue_queue_duration_seconds_bucket{job=~"kube-controller-manager", name=~"deployment|replicaset|endpoint|endpoint_slice|endpoint_slice_mirroring"}[5m])))
```

![[Pasted image 20250824235701.png]]

![[Pasted image 20250824235715.png]]
![[Pasted image 20250824235731.png]]

## Work queue depth
- How many actions in the workqueue are waiting to be processed.
- 다음 쿼리를 사용하여 작업 대기열의 깊이를 검색합니다. 이 메트릭은 대기열에 나열된 작업의 수입니다:
```bash
#
workqueue_depth
workqueue_depth{job="kube-controller-manager"}
rate(workqueue_depth{job="kube-controller-manager"}[5m])

#
sum(rate(workqueue_depth{job="kube-controller-manager"}[5m])) by (instance, name)
```

![[Pasted image 20250824235800.png]]
![[Pasted image 20250824235808.png]]

- 이상적인 값은 낮아야 합니다(10 미만을 권장). 대기열 길이가 너무 길고 상태가 오래 지속되면 동기화를 위해 대기 중인 서비스가 작업 대기열에 많이 존재한다는 것을 의미합니다.
- 이를 해결하려면 클러스터에서 노드, 포드 및 서비스의 변경 빈도를 적절히 줄여야 합니다. 위의 결과는 병렬로 많은 수의 배포를 생성하기 위한 시뮬레이션입니다.
- 작업 대기열의 피크가 있지만. 길이는 10 이상이지만 곧 정상으로 돌아오므로 문제가 되지 않습니다. 작업 대기열 등에 대한 자세한 메트릭은 Kubernetes 메트릭 참조에서 확인할 수 있습니다

# K8S Blog : 성능 관련 정보
## 2025
- Kubernetes v1.34 Sneak Peek : 8월 27일 릴리즈 - [Link](https://kubernetes.io/blog/2025/07/28/kubernetes-v1-34-sneak-peek/)
- **Announcing etcd v3.6.0** : 기존 v3.5 대비 평균 메모리 사용량 최소 50% 줄음, 처리량은 10% 향상 - [Link](https://kubernetes.io/blog/2025/05/15/announcing-etcd-3.6/) , [Release](https://github.com/etcd-io/etcd/blob/main/CHANGELOG/CHANGELOG-3.6.md) ⇒ _**etcd 3.6 버전 확인!**_
    - **etcd Robustness Testing** : the robustness testing framework for etcd, a distributed key-value store - [Github](https://github.com/etcd-io/etcd/tree/main/tests/robustness)
![[Pasted image 20250824235852.png]]
- Kubernetes v1.33: Updates to Container Lifecycle - [Link](https://kubernetes.io/blog/2025/05/14/kubernetes-v1-33-updates-to-container-lifecycle/)
- Kubernetes v1.33: Image Pull Policy the way you always thought it worked! - [Link](https://kubernetes.io/blog/2025/05/12/kubernetes-v1-33-ensure-secret-pulled-images-alpha/)
    - 원리부터 파악하는 컨테이너 이미지 PULL (w/ curl) - [Blog](https://iwanhae.tistory.com/m/3)
- Kubernetes v1.33: **Streaming List responses** : **1GB 데이터 x 10개 List 요청** 시 메모리 사용량이 70~80GB → 3GB로 1/20 감소 - [Link](https://kubernetes.io/blog/2025/05/09/kubernetes-v1-33-streaming-list-responses/)

![[Pasted image 20250824235926.png]]

- Kubernetes v1.33: **Image Volumes** graduate to **beta**! - [Link](https://kubernetes.io/blog/2025/04/29/kubernetes-v1-33-image-volume-beta/) , [Docs](https://kubernetes.io/docs/tasks/configure-pod-container/image-volumes/)
    - Kubernetes 1.31: **Read Only Volumes** Based On **OCI** Artifacts (**alpha**) : OCI 호환 이미지를 파드의 볼륨 마운트 사용 - [Link](https://kubernetes.io/blog/2024/08/16/kubernetes-1-31-image-volume-source/)
- Kubernetes v1.33: Continuing the transition from Endpoints to EndpointSlices - [Link](https://kubernetes.io/blog/2025/04/24/endpoints-deprecation/)

## 2024
Enhancing Kubernetes API Server Efficiency with API Streaming : List 요청 처리 작업에 상당한 메모리 소비 초래 → api OOM - [Link](https://kubernetes.io/blog/2024/12/17/kube-apiserver-api-streaming/)
![[Pasted image 20250825000003.png]]
- WatchListClient (beta) 로 **List → Watch 로 전환**, Watch 요청은 읽기 작업의 확장성을 향상시키도록 설계된 인메모리 Watch Cache 에서 처리
- 테스트 시나리오 : 각각 **1MB의 데이터를 포함하는 400개의 Secrets을 생성하고, Informers를 사용하여 모든 Secrets을 검색**
    - api 서버가 기존 **20GB** 임시 **메모리** 사용 → **2GB**로 안정화 사용.
- Kubernetes v1.31: Accelerating Cluster Performance with **Consistent Reads from Cache** - [Link](https://kubernetes.io/blog/2024/08/15/consistent-read-from-cache-beta/)
    - 읽기 작업을 최적화하기 위해 워치 캐시를 사용. 데이터를 필터링하는 요청은 **캐시**에서 처리되고, etcd에서 최소한의 메타데이터만 읽어옴.
    - **30% reduction** in kube-apiserver CPU usage
    - **25% reduction** in etcd CPU usage
    - **Up to 3x reduction** (from 5 seconds to 1.5 seconds) in 99th percentile pod LIST request latency
- Image Filesystem: Configuring Kubernetes to store containers on a separate filesystem - [Link](https://kubernetes.io/blog/2024/01/23/kubernetes-separate-image-filesystem/)

# [Tuning] API 서버 flag : max-requests-inflight (기본값 400) , max-mutating-requests-inflight (기본값 200) - [Docs](https://aws.github.io/aws-eks-best-practices/ko/scalability/docs/control-plane/) , [APF](https://docs.ocha.ng/kubernetes/apf)

- API 서버는 `--max-requests-inflight` query-type(get,list,watch...)및 `--max-mutating-requests-inflight` mutating-type(create,update,delete...) 플래그로 지정된 값을 합산하여 허용할 수 있는 총 진행 중인 요청 수를 구성합니다.
- AWS EKS는 이러한 플래그에 대해 기본값인 400개 및 200개 요청을 사용하므로 주어진 시간에 총 600개의 요청을 전달할 수 있습니다.
- APF는 이러한 600개의 요청을 다양한 요청 유형으로 나누는 방법을 지정합니다.
- AWS EKS 컨트롤 플레인은 각 클러스터에 최소 2개의 API 서버가 등록되어 있어 가용성이 높습니다.
- 이렇게 하면 클러스터 전체의 총 진행 중인 요청 수가 1200개로 늘어납니다.

```bash
#
kubectl exec -it -n kube-system kube-apiserver-myk8s-control-plane -- kube-apiserver -h
...
      --max-mutating-requests-inflight int                                                                
                This and --max-requests-inflight are summed to determine the server's total concurrency limit (which
                must be positive) if --enable-priority-and-fairness is true. Otherwise, this flag limits the maximum
                number of mutating requests in flight, or a zero value disables the limit completely. (default 200)
      --max-requests-inflight int                                                                                
                This and --max-mutating-requests-inflight are summed to determine the server's total concurrency
                limit (which must be positive) if --enable-priority-and-fairness is true. Otherwise, this flag
                limits the maximum number of non-mutating requests in flight, or a zero value disables the limit
                completely. (default 400)
      --min-request-timeout int                                                                                  
                An optional field indicating the minimum number of seconds a handler must keep a request open before
                timing it out. Currently only honored by the watch request handler, which picks a randomized value
                above this number as the connection timeout, to spread out load. (default 1800)
      --request-timeout duration                                                                                 
                An optional field indicating the duration a handler must keep a request open before timing it out.
                This is the default request timeout for requests but may be overridden by flags such as
                --min-request-timeout for specific types of requests. (default 1m0s)
```

- `--max-requests-inflight` **클러스터 상태 조회**
    - 설명 : API 서버가 동시에 처리할 수 있는 non-mutating 요청(GET, LIST, WATCH 등)의 최대 수를 제한
    - 동작 : 요청이 들어오면 큐에 넣고, 현재 inflight 수가 최대값을 넘으면 429 Too Many Requests를 반환.
- `--max-mutating-requests-inflight` **클러스터 상태 변경**
    - 설명 : API 서버가 동시에 처리할 수 있는 mutating 요청(POST, PUT, PATCH, DELETE)의 최대 수를 제한
    - 동작 : Mutating 요청은 클러스터 상태를 변경하기 때문에, 너무 많으면 **etcd I/O와 API 서버 내부 lock**에 부하 → 서버 전체 성능 저하 ⇒ 요청이 최대값을 넘어가면 역시 429 반환.
- `PriorityAndFairness(P&F)` : API 서버가 요청을 PriorityLevel에 따라 큐잉, 각 PriorityLevel별 할당 큐 존재

![[Pasted image 20250825000128.png]]

# [Tuning] CoreDNS : Multi Socket Plugin 소개 - [Youtube](https://www.youtube.com/watch?v=W3f5Ks0j2Q8)
- **CoreDNS** : DNS and Service Discovery - [Docs](https://coredns.io/)
    - Flexibility DNS Server written in **Go**
    - Focus on service discovery
    - **Plugin** based architecture, easily extended
    - Default DNS server in Kubernetes
    - Support DNS, DNS over TLS, DNS over gRPC
- **MultiSocket** 👍🏻 : improved vertical scaling - [Docs](https://coredns.io/plugins/multisocket/)
    - _multisocket_ allows to start multiple servers that will listen on one port.
    - 멀티소켓을 사용하면 동일한 포트에서 청취할 서버의 수를 정의할 수 있습니다.
    - SO_REUSEPORT 소켓 옵션을 사용하면 동일한 주소와 포트에서 여러 청취 소켓을 열 수 있습니다. 이 경우 커널은 소켓 간에 들어오는 연결을 분산합니다.
    - 이 옵션을 활성화하면 여러 서버를 시작할 수 있어 CPU 코어가 많은 환경에서 CoreDNS의 처리량이 증가합니다.
    - 기존 CoreDNS 기본 설정 환경에서 CPU가 많아져도 QPS가 늘어나지 않음. → **2 CPU 경우 40k qps**
![[Pasted image 20250825000206.png]]
![[Pasted image 20250825000215.png]]

## multisocket plugin
![[Pasted image 20250825000230.png]]
![[Pasted image 20250825000238.png]]
![[Pasted image 20250825000246.png]]
![[Pasted image 20250825000257.png]]
# ETCD 재시작으로 인한 장애 분석 Case : net.ipv4.tcp_timestamps , net.ipv4.tcp_wan_timestamps - [Link](https://segmentfault.com/a/1190000043406041)
![[Pasted image 20250825000325.png]]
![[Pasted image 20250825000342.png]]
# Youtube : large cluster

- **Building Armada – Running Batch Jobs at Massive Scale on Kubernetes - Jamie Poole, G-Research - [Youtube](https://www.youtube.com/watch?v=B3WPxw3OUl4)**
- **Automated Multi-Cloud Large Scale K8s Cluster Lifecycle Management - Sourav Khandelwal, Databricks - [Youtube](https://www.youtube.com/watch?v=9E7lenP6pFc)**
- **An Alternative Metadata System for Large Kubernetes Clusters - Yingcai Xue & Yixiang Chen, ByteDance - [Youtube](https://www.youtube.com/watch?v=MGOa8Nn8_S0)**
- Large Scale Automated Storage with Kubernetes - Celina Ward, Software Engineer & Matt Schallert, Site Reliability Engineer, Uber - [Youtube](https://www.youtube.com/watch?v=aDFm5KaTaOk)
- **Large-Scale Practice of Persistent Memory in Alibaba Cloud - Junbao Kan & Qingcan Wang, Alibaba - [Youtube](https://www.youtube.com/watch?v=3MKbk7AS8Jw)**
- Managing Large-Scale Kubernetes Clusters Effectively and Reliably - Yong Zhang & Zhixian Lin - [Youtube](https://www.youtube.com/watch?v=yUL7NKt2hAY)
- **How to Stabilize a GenAI-First, Modern Data LakeHouse: Provision 20,000 Ephemeral Data Lakes/Year - [Youtube](https://www.youtube.com/watch?v=L02zdTo7HT0)**
- **Scaling Kubernetes Networking to 1k, 5k, ... 100k Nodes!? - Marcel Zięba, Isovalent & Dorde Lapcevic, Google - [Youtube](https://www.youtube.com/watch?v=VWGB-NW800Y)**
- Collecting Operational Metrics for a Cluster with 5,000 Namespaces - Rob Szumski & Chance Zibolski, Red Hat - [Youtube](https://www.youtube.com/watch?v=JHmWRBWPKog)
- Scaling Kubernetes Networking Beyond 100k Endpoints - Rob Scott & Minhan Xia, Google - [Youtube](https://www.youtube.com/watch?v=a6SfbeM06Qo)
- Per-Node Api-Server Proxy: Expand the Cluster's Scale and Stability - Weizhou Lan & Iceber Gu - [Youtube](https://www.youtube.com/watch?v=O7w7e2iA7lA)
- Build a High Performance Remote Storage for Prometheus with Unlimited Time Series - Yang Xiang - [Youtube](https://www.youtube.com/watch?v=7pQZyYEz1l4)
- **BGP Peering Patterns for Kubernetes Networking at Preferred Networks - Sho Shimizu & Yutaro Hayakawa - [Youtube](https://www.youtube.com/watch?v=n7_I4zu6f_M)**
- Kubernetes Performance Tuning Workshop | Anton Weiss - [Youtube](https://www.youtube.com/watch?v=7CObA_qY6vo)

# [sysdid] k8s 구성 모니터링 PromQL : kube-proxy - [Link](https://www.sysdig.com/blog/monitor-kube-proxy) , Kubelet - [Link](https://www.sysdig.com/blog/how-to-monitor-kubelet) , kube-controller-manager - [Link](https://www.sysdig.com/blog/how-to-monitor-kube-controller-manager) , CoreDNS - [Link](https://www.sysdig.com/blog/how-to-monitor-coredns)


# `도전과제2` kube-burner 벤치마크 실행 시 관련 메트릭 발생하여, 프로메테우스/그파라나에서 확인 설정 해보기 - [Docs](https://kube-burner.github.io/kube-burner/v1.17.1/observability/) , [Grafana](https://github.com/kube-burner/kube-burner/tree/main/examples/grafana-dashboards) , [Github](https://github.com/kube-burner/kube-burner/tree/main/docs/observability)

^c725e1

# `도전과제3` kube-burner 가 제공하는 다양한 workloads 벤치마크 실행 및 k8s 튜닝 해보기 - [Github](https://github.com/kube-burner/kube-burner/tree/main/examples/workloads)

^ae5f3d
