- **샘플 애플리케이션 배포 및 확인** : `cilium-dbg`
    - 샘플 애플리케이션 배포
```bash
# 샘플 애플리케이션 배포
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webpod
spec:
  replicas: 3
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

* 확인 : cilium-dbg
```bash
# 배포 확인
kubectl get deploy,svc,ep webpod -owide
kubectl get endpointslices -l app=webpod
kubectl get ciliumendpoints # IP 확인

#
kubectl exec -n kube-system ds/cilium -- cilium-dbg ip list
kubectl exec -n kube-system ds/cilium -- cilium-dbg endpoint list

kubectl exec -n kube-system ds/cilium -- cilium-dbg service list
20   10.96.32.212:80/TCP      ClusterIP      1 => 172.20.0.140:80/TCP (active)       
                                             2 => 172.20.1.213:80/TCP (active)       
                                             3 => 172.20.2.43:80/TCP (active) 

kubectl exec -n kube-system ds/cilium -- cilium-dbg bpf lb list
kubectl exec -n kube-system ds/cilium -- cilium-dbg bpf lb list | grep 10.96.32.212
10.96.32.212:80/TCP (0)        0.0.0.0:0 (20) (0) [ClusterIP, non-routable]   
10.96.32.212:80/TCP (3)        172.20.2.43:80/TCP (20) (3)                                   
10.96.32.212:80/TCP (2)        172.20.1.213:80/TCP (20) (2)                                                 
10.96.32.212:80/TCP (1)        172.20.0.140:80/TCP (20) (1) 

kubectl exec -n kube-system ds/cilium -- cilium-dbg bpf nat list

# map
kubectl exec -n kube-system ds/cilium -- cilium-dbg map list | grep -v '0             0'
Name                           Num entries   Num errors   Cache enabled
cilium_lb4_backends_v3         16            0            true
cilium_lb4_reverse_nat         20            0            true
cilium_runtime_config          256           0            true
cilium_policy_v2_01061         3             0            true
cilium_lb4_services_v2         45            0            true
cilium_ipcache_v2              22            0            true
cilium_lxc                     1             0            true
cilium_policy_v2_01990         2             0            true

kubectl exec -n kube-system ds/cilium -- cilium-dbg map get cilium_lb4_services_v2
kubectl exec -n kube-system ds/cilium -- cilium-dbg map get cilium_lb4_services_v2 | grep 10.96.32.212
Key                            Value                     State   Error
10.96.32.212:80/TCP (0)        0 3[0] (20) [0x0 0x0]             
10.96.32.212:80/TCP (1)        16 0[0] (20) [0x0 0x0]            
10.96.32.212:80/TCP (2)        14 0[0] (20) [0x0 0x0]            
10.96.32.212:80/TCP (3)        15 0[0] (20) [0x0 0x0]   

kubectl exec -n kube-system ds/cilium -- cilium-dbg map get cilium_lb4_backends_v3
kubectl exec -n kube-system ds/cilium -- cilium-dbg map get cilium_lb4_reverse_nat
kubectl exec -n kube-system ds/cilium -- cilium-dbg map get cilium_ipcache_v2
```

* 통신 확인 & hubble → 문제를 해결하려먼 어떻게 해야 될까? 방안1 (라우팅 설정 - 수동 or 자동 BGP), 방안2 (Overlay Network)
```bash
# 통신 확인 : 문제 확인
kubectl exec -it curl-pod -- curl webpod | grep Hostname
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s --connect-timeout 1 webpod | grep Hostname; echo "---" ; sleep 1; done'


# k8s-w0 노드에 배포된 webpod 파드 IP 지정
export WEBPOD=$(kubectl get pod -l app=webpod --field-selector spec.nodeName=k8s-w0 -o jsonpath='{.items[0].status.podIP}')
echo $WEBPOD

# 신규 터미널 [router]
tcpdump -i any icmp -nn

# 
kubectl exec -it curl-pod -- ping -c 2 -w 1 -W 1 $WEBPOD

# 신규 터미널 [router] : 라우팅이 어떻게 되는가?
tcpdump -i any icmp -nn
22:08:20.123415 eth1  In  IP 172.20.0.89 > 172.20.2.36: ICMP echo request, id 345, seq 1, length 64
22:08:20.123495 eth0  Out IP 172.20.0.89 > 172.20.2.36: ICMP echo request, id 345, seq 1, length 64

# 신규 터미널 [router]
ip -c route
ip route get 172.20.2.36
172.20.2.36 via 10.0.2.2 dev eth0 src 10.0.2.15 uid 0


#
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s --connect-timeout 1 webpod | grep Hostname; echo "---" ; sleep 1; done'

# 신규 터미널 [router]
tcpdump -i any tcp port 80 -nn


# hubble 확인
# hubble ui 웹 접속 주소 확인 : default 네임스페이스 확인
NODEIP=$(ip -4 addr show eth1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
echo -e "http://$NODEIP:30003"

# hubble relay 포트 포워딩 실행
cilium hubble port-forward&
hubble status

# flow log 모니터링
hubble observe -f --protocol tcp --pod curl-pod

```


# 도전과제1 직접 router 에 static route 설정해서 문제 해결 후 트래픽 확인해보세요. 실습 완료 후 해당 설정은 다시 삭제해주세요.

^475163

현재는 노드가 총 3대이지만, 노드가 100대 이상되고, PodCIDR가 변경될때는 어떻게 해야 될까요?