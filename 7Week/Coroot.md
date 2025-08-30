# Coroot 소개 : - [Github](https://github.com/coroot/coroot) , [Architecture](https://docs.coroot.com/installation/architecture)
![[Pasted image 20250830002515.png]]

- **Coroot-node-agent**
    - Coroot-node-agent는 eBPF로 구동되는 오픈 소스 관측 가능성 에이전트입니다.
    - 이 에이전트는 노드에서 실행되는 모든 컨테이너에서 메트릭, 로그, 추적 및 프로필을 수집합니다.
    - 에이전트는 메트릭에 대한 pull 모드와 push 모드를 모두 지원합니다.
    - 이 에이전트는 Prometheus 형식으로 메트릭을 노출하고 Prometheus Remote Write protocol을 사용하여 직접 Coroot로 메트릭을 전송할 수도 있습니다.
    - 로그와 추적은 OpenTelemetry 프로토콜을 통해 Coroot로 전송되며, 프로필은 custom HTTP 기반 프로토콜을 사용하여 전송됩니다.
    - 전체 커버리지를 보장하려면 클러스터의 모든 노드에 에이전트를 설치해야 합니다.
    - 쿠버네티스를 사용하는 경우 데몬셋으로 배포되므로 각 노드에 자동으로 추가됩니다.
- **Coroot-cluster-agent**
    - Coroot-cluster-agent는 클러스터 전반의 원격 측정 데이터를 수집하는 전용 도구입니다:
    - Coroot's Service Map을 통해 데이터베이스를 검색하여 데이터베이스 메트릭을 수집합니다.
    - 에이전트는 Coroot에서 제공하는 자격 증명을 사용하여 Postgres, MySQL, Redis, Memcached, MongoDB와 같은 식별된 데이터베이스에 연결하고 데이터베이스별 메트릭을 수집한 다음 Prometheus Remote Write protocol을 사용하여 Coroot에 전송합니다.
    - coroot-node-agent에 내장된 eBPF 기반 continuous profiler 외에도, Coroot는 애플리케이션 수준의 프로파일링도 지원합니다.
    - cluster-agent는 [coroot.com/profile-scrape](http://coroot.com/profile-scrape) 과 [coroot.com/profile-port](http://coroot.com/profile-port), 로 주석이 달린 Go 애플리케이션을 발견하고 인스턴스에서 CPU 및 메모리 프로파일을 수집할 수 있습니다.
- **OpenTelemetry**
- Prometheus :
- ClickHouse :

# Coroot Community Edition 설치 - [Docs](https://docs.coroot.com/) , [Requirements](https://docs.coroot.com/installation/requirements)

![[Pasted image 20250830002651.png]]

```bash
#
helm repo add coroot https://coroot.github.io/helm-charts
helm repo update coroot

# Install the Coroot Operator:
helm install -n coroot --create-namespace coroot-operator coroot/coroot-operator

# Install the Coroot Community Edition. This chart creates a minimal Coroot Custom Resource:
helm install -n coroot coroot coroot/coroot-ce \
  --set "clickhouse.shards=1,clickhouse.replicas=1,clickhouse.storage.size=20Gi"

# http://localhost:30001 접속
kubectl patch svc -n coroot coroot-coroot -p '{"spec": {"type": "NodePort", "ports": [{"port": 8080, "targetPort": 8080, "nodePort": 30001}]}}'
open http://localhost:30001

# http://localhost:30000 접속
kubectl patch svc -n coroot coroot-prometheus -p '{"spec": {"type": "NodePort", "ports": [{"port": 9090, "targetPort": 9090, "nodePort": 30002}]}}'
open http://localhost:30002     

# 아래 그림 처럼 API Key 확인 후 helm 업그레이드
helm upgrade --install -n coroot coroot coroot/coroot-ce --reuse-values \
--set "apiKey=TzW-LFf_RypdA2p9sGp3xIQRbPlC-Xn2"
```

![[Pasted image 20250830002710.png]]

