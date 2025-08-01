# 설치 전 확인
```bash
#
cilium status
cilium config view | grep -i hubble
kubectl get cm -n kube-system cilium-config -o json | jq

#
kubectl get secret -n kube-system | grep -iE 'cilium-ca|hubble'
ss -tnlp | grep -iE 'cilium|hubble' | tee before.txt
```


# Hubble 설치
```bash 
# 설치방안 1 : hubble 활성화, 메트릭 설정 등등
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
--set hubble.enabled=true \
--set hubble.relay.enabled=true \
--set hubble.ui.enabled=true \
--set hubble.ui.service.type=NodePort \
--set hubble.ui.service.nodePort=31234 \
--set hubble.export.static.enabled=true \
--set hubble.export.static.filePath=/var/run/cilium/hubble/events.log \
--set prometheus.enabled=true \
--set operator.prometheus.enabled=true \
--set hubble.metrics.enableOpenMetrics=true \
--set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"

# 설치방안 2 : hubble 활성화
cilium hubble enable
cilium hubble enable --ui


# This is required for Relay to operate correctly.
cilium status
Hubble Relay:       OK

#
cilium config view | grep -i hubble
kubectl get cm -n kube-system cilium-config -o json | grep -i hubble

#
kubectl get secret -n kube-system | grep -iE 'cilium-ca|hubble'

# # Enabling Hubble requires the TCP port 4244 to be open on all nodes running Cilium.
ss -tnlp | grep -iE 'cilium|hubble' | tee after.txt
vi -d before.txt after.txt
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo ss -tnlp |grep 4244 ; echo; done

#
kubectl get pod -n kube-system -l k8s-app=hubble-relay
kc describe pod -n kube-system -l k8s-app=hubble-relay

kc get svc,ep -n kube-system hubble-relay
...
NAME                     ENDPOINTS           AGE
endpoints/hubble-relay   172.20.1.202:4245   7m54s


# hubble-relay 는 hubble-peer 의 서비스(ClusterIP :443)을 통해 모든 노드의 :4244에 요청 가져올 수 있음
kubectl get cm -n kube-system
kubectl describe cm -n kube-system hubble-relay-config
...
cluster-name: default
peer-service: "hubble-peer.kube-system.svc.cluster.local.:443"
listen-address: :4245
...

#
kubectl get svc,ep -n kube-system hubble-peer
NAME                  TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/hubble-peer   ClusterIP   10.96.145.0   <none>        443/TCP   5h55m

NAME                    ENDPOINTS                                                     AGE
endpoints/hubble-peer   192.168.10.100:4244,192.168.10.101:4244,192.168.10.102:4244   5h55m

#
kc describe pod -n kube-system -l k8s-app=hubble-ui
...
Containers:
  frontend:
  ...
  backend:
...

kc describe cm -n kube-system hubble-ui-nginx
...

#
kubectl get svc,ep -n kube-system hubble-ui
NAME                TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
service/hubble-ui   NodePort   10.96.66.67   <none>        80:31234/TCP   17m

NAME                  ENDPOINTS          AGE
endpoints/hubble-ui   172.20.2.70:8081   17m

# hubble ui 웹 접속 주소 확인
NODEIP=$(ip -4 addr show eth1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
echo -e "http://$NODEIP:31234"

```

hubble ui 웹 접속 후 → kube-system 네임스페이스 선택
![[Pasted image 20250726001647.png]]

# Hubble Client 설치
```bash
# Linux 실습 환경에 설치 시
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
which hubble
hubble status
```

# Hubble API Access 
```bash
#
cilium hubble port-forward&
Hubble Relay is available at 127.0.0.1:4245

ss -tnlp | grep 4245

# Now you can validate that you can access the Hubble API via the installed CLI
hubble status
Healthcheck (via localhost:4245): Ok
Current/Max Flows: 12,285/12,285 (100.00%)
Flows/s: 41.20

# hubble (api) server 기본 접속 주소 확인
hubble config view 
...
port-forward-port: "4245"
server: localhost:4245
...

# (옵션) 현재 k8s-ctr 가상머신이 아닌, 자신의 PC에 kubeconfig 설정 후 아래 --server 옵션을 통해 hubble api server 사용해보자!
hubble help status | grep 'server string'
      --server string                 Address of a Hubble server. Ignored when --input-file or --port-forward is provided. (default "localhost:4245")

# You can also query the flow API and look for flows
kubectl get ciliumendpoints.cilium.io -n kube-system # SECURITY IDENTITY
hubble observe
hubble observe -h
hubble observe -f

```

# Hubble Exporter 
- Hubble Exporter는 나중에 사용할 수 있도록 **Hubble flows 로그를 파일**에 저장할 수 있는 cilium-agent의 기능입니다.
- Hubble Exporter는 file rotation, size limits, filters, field masks를 지원합니다.

## Hubble Exporter 설정
```bash
# 설정 : 아래 설정할 필요 없음
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
   --set hubble.enabled=true \
   --set hubble.export.static.enabled=true \
   --set hubble.export.static.filePath=/var/run/cilium/hubble/events.log

kubectl -n kube-system rollout status ds/cilium


# 확인
kubectl get cm -n kube-system cilium-config -o json | grep hubble-export
cilium config view | grep hubble-export
hubble-export-allowlist                           
hubble-export-denylist                            
hubble-export-fieldmask                           
hubble-export-file-max-backups   # number of rotated Hubble export files to keep. (default 5)
hubble-export-file-max-size-mb   # size in MB at which to rotate the Hubble export file. (default 10)
hubble-export-file-path          # file path of target log file. (default /var/run/cilium/hubble/events.log)

# Verify that flow logs are stored in target files
kubectl -n kube-system exec ds/cilium -- tail -f /var/run/cilium/hubble/events.log
kubectl -n kube-system exec ds/cilium -- sh -c 'tail -f /var/run/cilium/hubble/events.log' | jq
```


## Filters & Field mask
```bash
# You can use hubble CLI to generated required filters (see Specifying Raw Flow Filters for more examples).
# For example, to filter flows with verdict DENIED or ERROR, run:
hubble observe --verdict DROPPED --verdict ERROR --print-raw-filters
allowlist:
- '{"verdict":["DROPPED","ERROR"]}'


# To keep all information except pod labels:
hubble-export-fieldmask: time source.identity source.namespace source.pod_name destination.identity destination.namespace destination.pod_name source_service destination_service l4 IP ethernet l7 Type node_name is_reply event_type verdict Summary

# To keep only timestamp, verdict, ports, IP addresses, node name, pod name, and namespace:
hubble-export-fieldmask: time source.namespace source.pod_name destination.namespace destination.pod_name l4 IP node_name is_reply verdict


# 설정 방안 1 : Then paste the output to hubble-export-allowlist in cilium-config Config Map:
kubectl -n kube-system patch cm cilium-config --patch-file=/dev/stdin <<-EOF
data:
  hubble-export-allowlist: '{"verdict":["DROPPED","ERROR"]}'
  hubble-export-denylist: '{"source_pod":["kube-system/"]},{"destination_pod":["kube-system/"]}'
EOF

# 설정 방안 2 : helm 업그레이드
helm upgrade cilium cilium/cilium --version 1.17.6 \
   --set hubble.enabled=true \
   --set hubble.export.static.enabled=true \
   --set hubble.export.static.filePath=/var/run/cilium/hubble/events.log \
   --set hubble.export.static.allowList[0]='{"verdict":["DROPPED","ERROR"]}'
   --set hubble.export.static.denyList[0]='{"source_pod":["kube-system/"]}' \
   --set hubble.export.static.denyList[1]='{"destination_pod":["kube-system/"]}' \
   --set "hubble.export.static.fieldMask={time,source.namespace,source.pod_name,destination.namespace,destination.pod_name,l4,IP,node_name,is_reply,verdict,drop_reason_desc}"


# 확인
cilium config view | grep hubble-export
hubble-export-allowlist                           {"verdict":["DENIED","ERROR"]}
...

kubectl -n kube-system exec ds/cilium -- tail -f /var/run/cilium/hubble/events.log
{"flow":{"time":"2023-08-21T12:12:13.517394084Z","verdict":"DROPPED","IP":{"source":"fe80::64d8:8aff:fe72:fc14","destination":"ff02::2","ipVersion":"IPv6"},"l4":{"ICMPv6":{"type":133}},"source":{},"destination":{},"node_name":"kind-kind/kind-worker","drop_reason_desc":"INVALID_SOURCE_IP"},"node_name":"kind-kind/kind-worker","time":"2023-08-21T12:12:13.517394084Z"}
{"flow":{"time":"2023-08-21T12:12:18.510175415Z","verdict":"DROPPED","IP":{"source":"10.244.1.60","destination":"10.244.1.5","ipVersion":"IPv4"},"l4":{"TCP":{"source_port":44916,"destination_port":80,"flags":{"SYN":true}}},"source":{"namespace":"default","pod_name":"xwing"},"destination":{"namespace":"default","pod_name":"deathstar-7848d6c4d5-th9v2"},"node_name":"kind-kind/kind-worker","drop_reason_desc":"POLICY_DENIED"},"node_name":"kind-kind/kind-worker","time":"2023-08-21T12:12:18.510175415Z"}
```

## Dynamic exporter configuration

- Standard hubble exporter configuration은 only one set of filters 허용하며 구성을 변경하려면 only one set of filters이 필요합니다.
- Dynamic flow logs를 사용하면 여러 필터를 동시에 구성하고 출력을 별도의 파일에 저장할 수 있습니다. 또한 변경된 구성을 적용하기 위해 실륨 포드 재시작이 필요하지 않습니다.
- Dynamic Hubble Exporter는 Config Map 속성으로 활성화됩니다. 파일 경로 값을 hubble-flowlogs-config-path로 설정할 때까지 비활성화됩니다.
```bash
# 설정 
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
   --set hubble.enabled=true \
   --set hubble.export.static.enabled=false \
   --set hubble.export.dynamic.enabled=true

kubectl -n kube-system rollout status ds/cilium


# 포드를 재시작할 필요 없이 흐름 로그 설정을 변경할 수 있습니다(구성 맵 전파 지연으로 인해 변경 사항을 60초 이내에 반영해야 함):
helm upgrade cilium cilium/cilium --version 1.17.6 \
   --set hubble.enabled=true \
   --set hubble.export.dynamic.enabled=true \
   --set hubble.export.dynamic.config.content[0].name=system \
   --set hubble.export.dynamic.config.content[0].filePath=/var/run/cilium/hubble/events-system.log \
   --set hubble.export.dynamic.config.content[0].includeFilters[0].source_pod[0]='kube_system/' \
   --set hubble.export.dynamic.config.content[0].includeFilters[1].destination_pod[0]='kube_system/'
```

# Configure TLS with Hubble 
https://docs.cilium.io/en/stable/observability/hubble/configuration/tls/

