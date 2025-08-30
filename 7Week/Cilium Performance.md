# Deep Dive Into Cilium Resilient Architecture : Map Size - Jussi Mäki & Martynas Pumputis, Isovalent - [Youtube](https://www.youtube.com/watch?v=YX0sql_3dt8)

## Drop 발생 중 : 메트릭 Service bacnend not found
![[Pasted image 20250826151114.png]]

## 일부 cilium-agent 상태 저하
![[Pasted image 20250826151131.png]]
## 문제 원인 발견! : BPF map pressure 가 144%!
![[Pasted image 20250826151150.png]]

## lb4_service_v2 Map 업데이트 담당하는 조정자가 failed 확인!
```bash
kubectl exec -it -n kube-system ds/cilium -- cilium status --verbose
```
![[Pasted image 20250826151225.png]]

## Service Map 상태 정보 확인 : Error: map is full 확인
```bash
# 현재는 명령 입력 방식이 변경됨 
k exec -it -n kube-system ds/cilium -- cilium statedb service4
```

![[Pasted image 20250826151301.png]]

## 해결 방안 : service map 크기를 늘리거나, 혹은 기존 service 를 삭제

![[Pasted image 20250826151321.png]]


# Deep Dive Into Cilium Resilient Architecture : API(Event) 와 Cilium-Agent(StateDB) 와 BPF 동작

## Cilium under the hood
![[Pasted image 20250826151355.png]]

## 장애 발생 가능 지점 → 실제로 장애 발생 시 영향도와 대응법?
![[Pasted image 20250826151415.png]]

## k8s (api) 이벤트가 발생하여 service BPF map 업데이트가 필요한 상황
![[Pasted image 20250826151429.png]]

## map 이 가득 차있을 경우 업데이트 할 수 없음 → 결국 재시도 로직이 필요함 ⇒ 이를 위해 결국 저장소(StateDB)가 필요함!
![[Pasted image 20250826151452.png]]

## 이를 위해, 중간 구성요소를 배치(DB, Reconciler) : 확인 방법, 메트릭 제공 등
![[Pasted image 20250826151510.png]]

- **StateDB**: **in-memory** database for state
    - In-memory transactional database
    - **Immutable** revisioned objects indexed in immutable radix trees
    - Channel-based change notification mechanism
![[Pasted image 20250826151542.png]]

## StateDB Reconciler sms
![[Pasted image 20250826151609.png]]
## map 업데이트 실패 시 : ENOSPC → 업데이트 메서드 실패가 위쪽으로 전파 → 재시도 동작
![[Pasted image 20250826151629.png]]
## 실패 상태와 메트릭 제공
![[Pasted image 20250826151651.png]]

## Reconciler 는 요청 처리 상태 실패(Error)을 전달하여 StateDB에 실패 상태로 저장
![[Pasted image 20250826151713.png]]

![[Pasted image 20250826151724.png]]
- Summary
    - Infrastructure is ready (StateDB, Hive and Agent Health, Reconciler)
    - In Cilium v1.16 we’ll see first use-cases, for example runtime device reconfiguration.
    - New features highly encounraged to leverage the new tooling for improved resiliency and observability.

# eBPF Maps : mapDynamicSizeRatio - [Docs](https://docs.cilium.io/en/stable/operations/performance/tuning/#ebpf-map-sizing) , [Maps-Limits](https://docs.cilium.io/en/stable/network/ebpf/maps/#bpf-map-limitations)
- All eBPF maps are created with upper capacity limits. Insertion beyond the limit would fail or constrain the scalability of the datapath.
- Cilium is using **auto**-derived defaults based on the given **ratio** of the **total system memory. → 메모리 비중 기준**
- However, the upper capacity limits used by the Cilium agent can be overridden for advanced users. Please refer to the [eBPF Maps](https://docs.cilium.io/en/stable/network/ebpf/maps/#bpf-map-limitations) guide.
```bash
#
kubectl exec -it -n kube-system ds/cilium -- cilium status --verbose
...
BPF Maps:   dynamic sizing: on (ratio: 0.002500)
  Name                          Size
  Auth                          524288
  Non-TCP connection tracking   65536
  TCP connection tracking       131072
  Endpoints                     65535
  IP cache                      512000
  IPv4 masquerading agent       16384
  IPv6 masquerading agent       16384
  IPv4 fragmentation            8192
  IPv4 service                  65536
  IPv6 service                  65536
  IPv4 service backend          65536
  IPv6 service backend          65536
  IPv4 service reverse NAT      65536
  IPv6 service reverse NAT      65536
  Metrics                       1024
  Ratelimit metrics             64
  NAT                           131072
  Neighbor table                131072
  Endpoint policy               16384
  Policy stats                  65536
  Session affinity              65536
  Sock reverse NAT              65536
  
#
helm upgrade cilium cilium/cilium --version 1.18.1 --namespace kube-system --reuse-values \
--set bpf.distributedLRU.enabled=true --set bpf.mapDynamicSizeRatio=0.01

kubectl -n kube-system rollout restart ds/cilium


# BPF Maps 에 많은 값들이 노드 총 메모리 비중별로 반영
kubectl exec -it -n kube-system ds/cilium -- cilium status --verbose
...
BPF Maps:   dynamic sizing: on (ratio: 0.010000)
  Name                          Size
  Auth                          524288
  Non-TCP connection tracking   219824
  TCP connection tracking       439648
  Endpoints                     65535
  IP cache                      512000
  IPv4 masquerading agent       16384
  IPv6 masquerading agent       16384
  IPv4 fragmentation            8192
  IPv4 service                  65536
  IPv6 service                  65536
  IPv4 service backend          65536
  IPv6 service backend          65536
  IPv4 service reverse NAT      65536
  IPv6 service reverse NAT      65536
  Metrics                       1024
  Ratelimit metrics             64
  NAT                           439648
  Neighbor table                439648
  Endpoint policy               16384
  Policy stats                  65536
  Session affinity              65536
  Sock reverse NAT              219824

```

The following table presents the value that kube-proxy and Cilium sets for their own connection tracking tables when Cilium is configured with --bpf-map-dynamic-size-ratio: 0.0025

| vCPU | Memory (GiB) | Kube-proxy CT entries | Cilium CT entries |
| ---- | ------------ | --------------------- | ----------------- |
| 1    | 3.75         | 131072                | 131072            |
| 2    | 7.5          | 131072                | 131072            |
| 4    | 15           | 131072                | 131072            |
| 8    | 30           | 262144                | 284560            |
| 16   | 60           | 524288                | 569120            |
| 32   | 120          | 1048576               | 1138240           |
| 64   | 240          | 2097152               | 2276480           |
|  96  | 360          | 3145728               | 4552960           |


# Kubernetes API server improvements : v1.18 부터 Cilium_client 가 K8S API 서버 부하분산/HA 연결 가능 - [Blog](https://isovalent.com/blog/post/cilium-1-18/#operations)

- **Replacing kube-proxy** requires the **Cilium agen**t to communicate with the Kubernetes **API server** so that it is made aware of changes within the cluster (such as **new** or updated **services** and **endpoints**).
- **Prior to 1.18**, the Cilium agent could only be configured to speak with **a single instance of the API serv**er. Unfortunately, in most production environments, multiple API servers are deployed to provide high availability.
- To provide the required High Availability in the event of an API server failure, a new flag (`--k8s-api-server-urls)` has been added that allows a user to specify a number of Kubernetes API servers.
- Cilium finds a working API server from this list for an initial conversation, and then changes over to using the internal Kubernetes API service address which has **load balancing** and **high availability**.

![[Pasted image 20250826160443.png]]

```bash
#
kubectl exec -it -n kube-system ds/cilium -- cilium-agent -h
...
--enable-k8s                                                Enable the k8s clientset (default true)
--enable-k8s-api-discovery                                  Enable discovery of Kubernetes API groups and resources with the discovery API
--k8s-api-server-urls strings                               Kubernetes API server URLs
--k8s-client-burst int                                      Burst value allowed for the K8s client (default 20)
--k8s-client-connection-keep-alive duration                 Configures the keep alive duration of K8s client connections. K8 client is disabled if the value is set to 0 (default 30s)
--k8s-client-connection-timeout duration                    Configures the timeout of K8s client connections. K8s client is disabled if the value is set to 0 (default 30s)
--k8s-client-qps float32                                    Queries per second limit for the K8s client (default 10)
--k8s-heartbeat-timeout duration                            Configures the timeout for api-server heartbeat, set to 0 to disable (default 30s)
...
```

- 일부 시나리오에서는 Kubernetes API 서버에서 정보를 검색하면 과부하가 걸리거나 API 서버의 가용성에 해로울 수 있습니다(특히 LIST 요청).
- API 서버에 요청 시 exponential back-off 설정 가능.

```bash
k8sClientExponentialBackoff:
    enabled: true
    backoffBaseSeconds: 1
    backoffMaxDurationSeconds: 120
```

