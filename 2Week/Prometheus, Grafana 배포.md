# Sample APP 배포
```bash
# 샘플 애플리케이션 배포
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webpod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webpod
  template:
    metadata:
      labels:
        app: webpod
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - sample-app
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: webpod
        image: traefik/whoami
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: webpod
  labels:
    app: webpod
spec:
  selector:
    app: webpod
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
EOF


# k8s-ctr 노드에 curl-pod 파드 배포
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: curl-pod
  labels:
    app: curl
spec:
  nodeName: k8s-ctr
  containers:
  - name: curl
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
EOF
```

# Sample APP 확인
```bash
# 배포 확인
kubectl get deploy,svc,ep webpod -owide
kubectl get endpointslices -l app=webpod
kubectl get ciliumendpoints
kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium-dbg endpoint list

# 통신 확인
kubectl exec -it curl-pod -- curl webpod | grep Hostname
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s webpod | grep Hostname; sleep 1; done'
```

# Install Prometheus & Grafana 
https://docs.cilium.io/en/stable/observability/grafana/

```bash
#
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/kubernetes/addons/prometheus/monitoring-example.yaml
configmap/grafana-config created
configmap/grafana-cilium-dashboard created
configmap/grafana-cilium-operator-dashboard created
configmap/grafana-hubble-dashboard created
configmap/grafana-hubble-l7-http-metrics-by-workload created
configmap/prometheus created

#
kubectl get deploy,pod,svc,ep -n cilium-monitoring
kubectl get cm -n cilium-monitoring
NAME                                         DATA   AGE
grafana-cilium-dashboard                     1      113s
grafana-cilium-operator-dashboard            1      113s
grafana-config                               3      113s
grafana-hubble-dashboard                     1      113s
grafana-hubble-l7-http-metrics-by-workload   1      113s
prometheus                                   1      113s

# 프로메테우스 서버 설정
kc describe cm -n cilium-monitoring prometheus

# 그라파나 서버 설정
kc describe cm -n cilium-monitoring grafana-config

# 그파라나 대시보드들 주입을 위한 설정
kc describe cm -n cilium-monitoring grafana-cilium-dashboard
kc describe cm -n cilium-monitoring grafana-hubble-dashboard
...
```

# Deploy Cilium and Hubble with metrics enabled
- Cilium, Hubble, and Cilium Operator는 기본적으로 메트릭을 노출하지 않습니다.
- 이러한 서비스에 대한 메트릭을 활성화하면 이러한 구성 요소가 실행 중인 클러스터의 모든 노드에 각각 **9962, 9965, 9963** 포트가 열립니다.
- Cilium, Hubble, and Cilium Operator의 메트릭은 모두 다음 헬름 값으로 서로 독립적으로 활성화할 수 있습니다
    - `prometheus.enabled=true`: Enables metrics for `cilium-agent`.
    - `operator.prometheus.enabled=true`: Enables metrics for `cilium-operator`.
    - `hubble.metrics.enabled`: Enables the provided list of Hubble metrics.
        - For Hubble metrics to work, Hubble itself needs to be enabled with `hubble.enabled=true`.
        - See [Hubble exported metrics](https://docs.cilium.io/en/stable/observability/metrics/#hubble-exported-metrics) for the list of available Hubble metrics.

```bash
# 이미 설정되어 있음
helm install cilium cilium/cilium --version 1.17.6 \
   --namespace kube-system \
   --set prometheus.enabled=true \
   --set operator.prometheus.enabled=true \
   --set hubble.enabled=true \
   --set hubble.metrics.enableOpenMetrics=true \
   --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"

# 호스트에 포트 정보 확인
ss -tnlp | grep -E '9962|9963|9965'
LISTEN 0      4096                *:9963             *:*    users:(("cilium-operator",pid=1488,fd=7)) # cilium-opeator 메트릭        
LISTEN 0      4096                *:9962             *:*    users:(("cilium-agent",pid=1894,fd=7))    # cilium 메트릭  
LISTEN 0      4096                *:9965             *:*    users:(("cilium-agent",pid=1894,fd=40))   # hubble 메트릭

for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo ss -tnlp | grep -E '9962|9963|9965' ; echo; done

```

# hostPC에서 접속을 위한 NodePort 설정 및 ‘프로메테우스 & 그라파나’ 웹 접속 확인

```bash
#
kubectl get svc -n cilium-monitoring
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
grafana      ClusterIP   10.96.212.137   <none>        3000/TCP   6m36s
prometheus   ClusterIP   10.96.240.147   <none>        9090/TCP   6m36s

# NodePort 설정
kubectl patch svc -n cilium-monitoring prometheus -p '{"spec": {"type": "NodePort", "ports": [{"port": 9090, "targetPort": 9090, "nodePort": 30001}]}}'
kubectl patch svc -n cilium-monitoring grafana -p '{"spec": {"type": "NodePort", "ports": [{"port": 3000, "targetPort": 3000, "nodePort": 30002}]}}'

# 확인
kubectl get svc -n cilium-monitoring
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
grafana      NodePort   10.96.212.137   <none>        3000:30002/TCP   14m
prometheus   NodePort   10.96.240.147   <none>        9090:30001/TCP   14m

# 접속 주소 확인
echo "http://192.168.10.100:30001"  # prometheus
echo "http://192.168.10.100:30002"  # grafana
```

# Prometheus
기본(쿼리창) : cilium_ , cilium_operator_ , hubble_

![[Pasted image 20250727010948.png]]

# Grafana
Cilium Metrics 대시보드 & 간단 쿼리문 알아보기! : Generic, API, Cilium(BPF, kvstore, NW info, Endpoints, k8s integration)

![[Pasted image 20250727011059.png]]

## Query
- **cilium_bpf_map_ops_total** : Number of **eBPF map operations performed**. ⇒ 수행된 eBPF Map 작업 수
    - `mapName` is deprecated and will be removed in 1.10. Use `map_name` instead.
```bash
topk(5, avg(rate(cilium_bpf_map_ops_total{k8s_app="cilium", pod=~"$pod"}[5m])) by (pod, map_name, operation))


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