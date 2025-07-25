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

