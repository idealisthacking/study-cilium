## Flannel CNI 설치 및 확인 - [Github](https://github.com/flannel-io/flannel)

### 설치 전 확인
```bash
kubectl cluster-info dump | grep -m 2 -E "cluster-cidr|service-cluster-ip-range"
"--service-cluster-ip-range=10.96.0.0/16",
"--cluster-cidr=10.244.0.0/16",

kubectl get pod -n kube-system -l k8s-app=kube-dns -owide
NAME                       READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
coredns-674b8bbfcf-fl4tr   0/1     Pending   0          39m   <none>   <none>   <none>           <none>
coredns-674b8bbfcf-npvs9   0/1     Pending   0          39m   <none>   <none>   <none>           <none>
        
ip -c link
ip -c route
brctl show

iptables-save
iptables -t nat -S
iptables -t filter -S
iptables -t mangle -S

tree /etc/cni/net.d/
        
```

### Flannel CNI 설치 - [Github](https://github.com/flannel-io/flannel)
```bash
# Needs manual creation of namespace to avoid helm error
kubectl create ns kube-flannel
kubectl label --overwrite ns kube-flannel pod-security.kubernetes.io/enforce=privileged

helm repo add flannel https://flannel-io.github.io/flannel/
helm repo list
helm search repo flannel
helm show values flannel/flannel

# k8s 관련 트래픽 통신 동작하는 nic 지정
cat << EOF > flannel-values.yaml
podCidr: "10.244.0.0/16"

flannel:
  args:
  - "--ip-masq"
  - "--kube-subnet-mgr"
  - "--iface=eth1"  
EOF

# helm 설치
helm install flannel --namespace kube-flannel flannel/flannel -f flannel-values.yaml
helm list -A

# 확인 : install-cni-plugin, install-cni
kc describe pod -n kube-flannel -l app=flannel

tree /opt/cni/bin/ # flannel
tree /etc/cni/net.d/
cat /etc/cni/net.d/10-flannel.conflist | jq
kc describe cm -n kube-flannel kube-flannel-cfg
...
net-conf.json:
----
{
  "Network": "10.244.0.0/16",
  "EnableNFTables": false,
  "Backend": {
    "Type": "vxlan"
  }
}

# 설치 전과 비교해보자
ip -c link
ip -c route | grep 10.244.
10.244.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1 
10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink 
10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink 

ping -c 1 10.244.1.0
ping -c 1 10.244.2.0

brctl show
iptables-save
iptables -t nat -S
iptables -t filter -S

# k8s-w1, k8s-w2 정보 확인
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i ip -c link ; echo; done
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i ip -c route ; echo; done
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i brctl show ; echo; done
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i sudo iptables -t nat -S ; echo; done
```

### 샘플 애플리케이션 배포 

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
      image: alpine/curl
      command: ["sleep", "36000"]
EOF

#
crictl ps
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo crictl ps ; echo; done


```

### 배포 확인

```bash
# 배포 확인
kubectl get deploy,svc,ep webpod -owide
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES           SELECTOR
deployment.apps/webpod   2/2     2            2           4m11s   webpod       traefik/whoami   app=webpod

NAME             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE     SELECTOR
service/webpod   ClusterIP   10.96.63.96   <none>        80/TCP    4m11s   app=webpod

NAME               ENDPOINTS                     AGE
endpoints/webpod   10.244.1.2:80,10.244.2.2:80   4m11s

#
kubectl api-resources | grep -i endpoint
endpoints                           ep           v1                                true         Endpoints
endpointslices                                   discovery.k8s.io/v1               true         EndpointSlice

kubectl get endpointslices -l app=webpod
NAME           ADDRESSTYPE   PORTS   ENDPOINTS               AGE
webpod-gt8kw   IPv4          80      10.244.2.2,10.244.1.2   5m18s


# 배포 전과 비교해보자
ip -c link
brctl show
iptables-save
iptables -t nat -S

# k8s-w1, k8s-w2 정보 확인
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i ip -c link ; echo; done
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i ip -c route ; echo; done
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i brctl show ; echo; done
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i sudo iptables -t nat -S ; echo; done

```

### 통신 확인 
```bash

#
kubectl get pod -l app=webpod -owide
POD1IP=10.244.1.2
kubectl exec -it curl-pod -- curl $POD1IP

#
kubectl get svc,ep webpod
kubectl exec -it curl-pod -- curl webpod
kubectl exec -it curl-pod -- curl webpod | grep Hostname
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s webpod | grep Hostname; sleep 1; done'

# Service 동작 처리에 iptables 규칙 활용 확인 >> Service 가 100개 , 1000개 , 10000개 증가 되면???
kubectl get svc webpod -o jsonpath="{.spec.clusterIP}"
SVCIP=$(kubectl get svc webpod -o jsonpath="{.spec.clusterIP}")
iptables -t nat -S | grep $SVCIP
for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i sudo iptables -t nat -S | grep $SVCIP ; echo; done
-A KUBE-SERVICES -d 10.96.255.104/32 -p tcp -m comment --comment "default/webpod cluster IP" -m tcp --dport 80 -j KUBE-SVC-CNZCPOCNCNOROALA
-A KUBE-SVC-CNZCPOCNCNOROALA ! -s 10.244.0.0/16 -d 10.96.255.104/32 -p tcp -m comment --comment "default/webpod cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ

```
