zerotay - [Link](https://zerotay-blog.vercel.app/6.PUBLISHED/Cilium%20%EA%B3%B5%EC%8B%9D%20%EB%AC%B8%EC%84%9C%20%ED%95%B8%EC%A6%88%EC%98%A8%20%EC%8A%A4%ED%84%B0%EB%94%94/2W%20-%20%ED%94%84%EB%A1%9C%EB%A9%94%ED%85%8C%EC%9A%B0%EC%8A%A4%EC%99%80%20%EA%B7%B8%EB%9D%BC%ED%8C%8C%EB%82%98%EB%A5%BC%20%ED%99%9C%EC%9A%A9%ED%95%9C%20%EB%AA%A8%EB%8B%88%ED%84%B0%EB%A7%81/)

# kube-prometheus-stack 설치
```bash
# 기존 프로메테우스, 그라파나 삭제
kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/1.18.0/examples/kubernetes/addons/prometheus/monitoring-example.yaml

# repo 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# 파라미터 파일 생성
cat <<EOT > monitor-values.yaml
prometheus:
  prometheusSpec:
    scrapeInterval: "15s"
    evaluationInterval: "15s"
  service:
    type: NodePort
    nodePort: 30001

grafana:
  defaultDashboardsTimezone: Asia/Seoul
  adminPassword: prom-operator
  service:
    type: NodePort
    nodePort: 30002

alertmanager:
  enabled: false
defaultRules:
  create: false
prometheus-windows-exporter:
  prometheus:
    monitor:
      enabled: false
EOT
cat monitor-values.yaml
```

```bash
# 배포
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 75.15.1 \
-f monitor-values.yaml --create-namespace --namespace monitoring


# 각각 웹 접속 실행
## Windows(WSL2) 사용자는 아래 주소를 자신의 웹 브라우저에서 기입 후 직접 접속, 이후에도 동일.
open http://127.0.0.1:30001 # macOS
open http://127.0.0.1:30002 # macOS

# 확인
## grafana : 프로메테우스는 메트릭 정보를 저장하는 용도로 사용하며, 그라파나로 시각화 처리
## prometheus-0 : 모니터링 대상이 되는 파드는 ‘exporter’라는 별도의 사이드카 형식의 파드에서 모니터링 메트릭을 노출, pull 방식으로 가져와 내부의 시계열 데이터베이스에 저장
## node-exporter : 노드익스포터는 물리 노드에 대한 자원 사용량(네트워크, 스토리지 등 전체) 정보를 메트릭 형태로 변경하여 노출
## operator : 시스템 경고 메시지 정책(prometheus rule), 애플리케이션 모니터링 대상 추가 등의 작업을 편리하게 할수 있게 CRD 지원
## kube-state-metrics : 쿠버네티스의 클러스터의 상태(kube-state)를 메트릭으로 변환하는 파드
helm list -n monitoring
kubectl get pod,svc,ingress,pvc -n monitoring
kubectl get-all -n monitoring
kubectl get prometheus,servicemonitors -n monitoring
kubectl get crd | grep monitoring

# 프로메테우스 버전 확인
kubectl exec -it sts/prometheus-kube-prometheus-stack-prometheus -n monitoring -c prometheus -- prometheus --version


# 삭제
helm uninstall -n monitoring kube-prometheus-stack
```

# Cilium 관련 메트릭용 Service 생성
```bash
#
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: cilium-agent-metrics
  namespace: kube-system
  labels:
    k8s-app: cilium-agent
spec:
  selector:
    k8s-app: cilium
  ports:
    - name: prometheus
      port: 9962
      targetPort: 9962     
EOF
kubectl get svc,ep -n kube-system cilium-agent-metrics
```

```bash
#
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: cilium-operator-metrics
  namespace: kube-system
  labels:
    k8s-app: cilium-operator
spec:
  selector:
    name: cilium-operator
  ports:
    - name: prometheus
      port: 9963
      targetPort: 9963
EOF
kubectl get svc,ep -n kube-system cilium-operator-metrics
```

```bash
#
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: hubble-metrics-service
  namespace: kube-system
  labels:
    k8s-app: hubble
spec:
  selector:
    k8s-app: cilium
  ports:
    - name: metrics
      port: 9965
      targetPort: 9965
EOF
kubectl get svc,ep -n kube-system hubble-metrics-service

```

# Prometheus 수집용 ServiceMonitor 생성 : Target 확인 → 프로메테우스 메트릭 수집 확인
```bash
#
cat << EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cilium-agent
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      k8s-app: cilium-agent
  namespaceSelector:
    matchNames:
      - kube-system
  endpoints:
    - port: prometheus
      interval: 30s
EOF

#
cat << EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cilium-operator
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      k8s-app: cilium-operator
  namespaceSelector:
    matchNames:
      - kube-system
  endpoints:
    - port: prometheus
      interval: 30s
EOF

#
cat << EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: hubble
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      k8s-app: hubble
  namespaceSelector:
    matchNames:
      - kube-system
  endpoints:
    - port: metrics
      interval: 30s
EOF
```

![[Pasted image 20250803213209.png]]

# 도전과제6 아래 Target 에 대한 문제를 해결해서 정상 수집하게 하고, Cilium 제공 그라파나 대시보드 - [Github](https://github.com/cilium/cilium/tree/main/examples/kubernetes/addons/prometheus/files/grafana-dashboards) or 공개 대시보드 추가해보자 - [Grafana-Search](https://grafana.com/grafana/dashboards/?search=cilium)

^689d0f

