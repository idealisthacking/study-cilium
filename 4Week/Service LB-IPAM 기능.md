# Service LB-IPAM 기능
# Service 추가 시 동작 : netshoot-web

```bash
#
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netshoot-web
  labels:
    app: netshoot-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: netshoot-web
  template:
    metadata:
      labels:
        app: netshoot-web
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: netshoot
          image: nicolaka/netshoot
          ports:
            - containerPort: 8080
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          command: ["sh", "-c"]
          args:
            - |
              while true; do 
                { echo -e "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\nOK from \$POD_NAME"; } | nc -l -p 8080 -q 1;
              done
EOF
```

```bash
#
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: netshoot-web
  labels:
    app: netshoot-web
spec:
  type: LoadBalancer
  selector:
    app: netshoot-web
  ports:
    - name: http
      port: 80      
      targetPort: 8080
EOF
```

```bash
#
kubectl get svc netshoot-web
NAME           TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
netshoot-web   LoadBalancer   10.96.64.29   192.168.10.212   80:30131/TCP   8s

#
cat << EOF | kubectl apply -f -
apiVersion: "cilium.io/v2alpha1"  # not v2
kind: CiliumL2AnnouncementPolicy
metadata:
  name: policy2
spec:
  serviceSelector:
    matchLabels:
      app: netshoot-web
  nodeSelector:
    matchExpressions:
      - key: kubernetes.io/hostname
        operator: NotIn
        values:
          - k8s-w0
  interfaces:
  - ^eth[1-9]+
  externalIPs: true
  loadBalancerIPs: true
EOF
```

```bash
# Service 별로 리더 노드가 다르다 : 즉, 외부 인입 시 Service 별로 나름 분산(?) 처리.. 
kubectl -n kube-system get lease | grep "cilium-l2announce"
cilium-l2announce-default-netshoot-web   k8s-w1                                                                     
cilium-l2announce-default-webpod         k8s-ctr    


# 호출 확인
## LBIP로 curl 요청 확인 : k8s 노드들에서 LB EXIP로 통신 가능!
kubectl get svc netshoot-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
LB2IP=$(kubectl get svc netshoot-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s $LB2IP

## 신규터미널 (router)
LB2IP=192.168.10.212
arping -i eth1 $LB2IP -c 2
curl -s $LB2IP

```

# Requesting IPs : 특정 Service에 EX-IP를 직접 설정 - [Docs](https://docs.cilium.io/en/stable/network/lb-ipam/#loadbalancerclass)

```bash
# Service netshoot-web 에 EX-IP를 직접 지정 변경
k9s → svc → <e> edit
 
## metadata.annotations 아래 아래 추가
  annotations:
    "lbipam.cilium.io/ips": "192.168.10.215"

#
kubectl get svc netshoot-web
NAME           TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
netshoot-web   LoadBalancer   10.96.64.29   192.168.10.215   80:30131/TCP   8m24s

#
kubectl get svc netshoot-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
LB2IP=$(kubectl get svc netshoot-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s $LB2IP

```

# Sharing Keys : EX-IP 1개를 각기 다른 Port 를 통해서 사용 - [Docs](https://docs.cilium.io/en/stable/network/lb-ipam/#requesting-ips)
```bash  
#
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: netshoot-web2
  labels:
    app: netshoot-web
spec:
  type: LoadBalancer
  selector:
    app: netshoot-web
  ports:
    - name: http
      port: 8080      
      targetPort: 8080
EOF
```

```bash
#
kubectl get svc -l app=netshoot-web
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
netshoot-web    LoadBalancer   10.96.64.29     192.168.10.215   80:30131/TCP     11m
netshoot-web2   LoadBalancer   10.96.191.204   192.168.10.212   8080:31946/TCP   8s

# Service netshoot-web, netshoot-web2 에 annotations 추가
k9s → svc → <e> edit 
## metadata.annotations 아래 아래 추가
  annotations:
    "lbipam.cilium.io/ips": "192.168.10.215"
    "lbipam.cilium.io/sharing-key": "1234"

#
kubectl get svc -l app=netshoot-web
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
netshoot-web    LoadBalancer   10.96.64.29     192.168.10.215   80:30131/TCP     14m
netshoot-web2   LoadBalancer   10.96.191.204   192.168.10.215   8080:31946/TCP   3m16s

# sharing-key 사용되는 IP는 모든 같은 리더 노드 사용
kubectl -n kube-system get lease | grep "cilium-l2announce"
cilium-l2announce-default-netshoot-web    k8s-ctr                                                                     7m44s
cilium-l2announce-default-netshoot-web2   k8s-ctr                                                                     4m22s
cilium-l2announce-default-webpod          k8s-ctr                                                                     34m


# 호출 확인
curl -s $LB2IP
curl -s $LB2IP:8080


# 신규터미널 (router)
LB2IP=192.168.10.215
arping -i eth1 $LB2IP -c 2
curl -s $LB2IP
curl -s $LB2IP:8080

```

# 도전과제5 Service 에 NodePort 비활성화 설정 해보세요 - Docs

^a89ab1
