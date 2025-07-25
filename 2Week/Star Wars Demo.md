![[Pasted image 20250726005523.png]]
- 스타워즈에서 영감을 받은 예제에서는 데스스타, 타이파이터, 엑스윙의 세 가지 마이크로서비스 애플리케이션이 있습니다.
- 데스스타는 포트 80에서 HTTP 웹서비스를 실행하며, 이 서비스는 두 개의 포드 복제본에 걸쳐 데스스타에 대한 요청을 로드 밸런싱하는 Kubernetes 서비스로 노출됩니다.
- 데스스타 서비스는 제국의 우주선에 착륙 서비스를 제공하여 착륙 포트를 요청할 수 있도록 합니다.
- 타이파이터 포드는 일반적인 제국 선박의 착륙 요청 클라이언트 서비스를 나타내며, 엑스윙은 동맹 선박의 유사한 서비스를 나타냅니다.
- 데스스타 착륙 서비스에 대한 접근 제어를 위한 다양한 보안 정책을 테스트할 수 있도록 존재합니다.

# 배포
```bash
#
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/http-sw-app.yaml
service/deathstar created
deployment.apps/deathstar created
pod/tiefighter created
pod/xwing created

# 파드 라벨 labels 확인
kubectl get pod --show-labels
NAME                        READY   STATUS    RESTARTS   AGE   LABELS
deathstar-8c4c77fb7-9klws   1/1     Running   0          29s   app.kubernetes.io/name=deathstar,class=deathstar,org=empire,pod-template-hash=8c4c77fb7
deathstar-8c4c77fb7-kkwds   1/1     Running   0          29s   app.kubernetes.io/name=deathstar,class=deathstar,org=empire,pod-template-hash=8c4c77fb7
tiefighter                  1/1     Running   0          29s   app.kubernetes.io/name=tiefighter,class=tiefighter,org=empire
xwing                       1/1     Running   0          29s   app.kubernetes.io/name=xwing,class=xwing,org=alliance

kubectl get deploy,svc,ep deathstar
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deathstar   2/2     2            2           114s

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/deathstar   ClusterIP   10.96.154.223   <none>        80/TCP    114s

NAME                  ENDPOINTS                      AGE
endpoints/deathstar   172.20.1.34:80,172.20.2.1:80   114s

#
kubectl get ciliumendpoints.cilium.io -A
kubectl get ciliumidentities.cilium.io

# in a multi-node installation, only the ones running on the same node will be listed
kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium endpoint list
c0 endpoint list
c1 endpoint list
c2 endpoint list # 현재 ingress/egress 에 정책(Policy) 없음! , Labels 정보 확인
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS (source:key[=value])
...
1579       Disabled           Disabled          318        k8s:app.kubernetes.io/name=deathstar                                                172.20.2.1    ready   
                                                           k8s:class=deathstar                                                                                       
                                                           k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default                                    
                                                           k8s:io.cilium.k8s.policy.cluster=default                                                                  
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                           
                                                           k8s:io.kubernetes.pod.namespace=default                                                                   
                                                           k8s:org=empire   

```