# eCHO Episode 194: Tetragon Everywhere
https://www.youtube.com/watch?v=JDvmlECqYg4

# Tetragon Resources - [Link](https://tetragon.io/docs/resources/)

# Overview : 실시간 eBPF 기반 보안 관찰 및 런타임 적용을 지원 - [Docs](https://tetragon.io/docs/overview/)
![[Pasted image 20250906225942.png]]
- **Tetragon**은 다음과 같은 보안에 중요한 이벤트를 감지하고 대응할 수 있습니다.
    - **Process** execution events
    - **System** call activity
    - **I/O** activity including network & file access
- Kubernetes 환경에서 Tetragon을 사용하면 Kubernetes를 인식합니다.
    - 즉, 네임스페이스, 포드 등의 Kubernetes ID를 이해하므로 개별 작업 부하와 관련하여 보안 이벤트 감지를 구성할 수 있습니다.

## **eBPF 실시간**

- Tetragon은 런타임 보안 강화 및 관찰 도구입니다. 즉, Tetragon은 **커널의 eBPF에 정책과 필터링을 직접 적용**합니다.
- 사용자 공간 에이전트로 이벤트를 전송하는 대신, **커널에서 직접 이벤트 필터링, 차단 및 대응**을 수행합니다.
- 관찰성 사용 사례의 경우, 커널에 직접 필터를 적용하면 관찰 오버헤드가 크게 줄어듭니다. 특히 전송, 읽기 또는 쓰기 작업과 같이 빈도가 높은 이벤트의 경우, **값비싼 컨텍스트 전환 및 웨이크업을 방지**함으로써 eBPF는 필요한 리소스를 절감합니다.
- 대신 Tetragon은 eBPF에서 **파일, 소켓, 바이너리 이름, 네임스페이스/기능 등 다양한 필터**를 제공하여 사용자가 특정 컨텍스트에서 중요하고 관련성 있는 이벤트를 지정하고 **해당 이벤트만 사용자 공간 에이전트로 전달**할 수 있도록 합니다.

## **eBPF 유연성**

- Tetragon은 L**inux 커널의 모든 함수에 연결**하여 인수, 반환 값, Tetragon이 **프로세스에 대해 수집하는 관련 메타데이터**(예: 실행 파일 이름), **파일** 및 기타 속성을 **필터링할** 수 있습니다.
- **추적 정책**을 작성함으로써 사용자는 다양한 보안 및 관측 가능성 사용 사례를 해결할 수 있습니다. 저장소에 이러한 사용 사례에 대한 여러 예제를 제공하고 있으며, 아래 '시작 가이드'에서 몇 가지를 강조하고 있지만, 사용자는 자신의 사용 사례에 맞는 새 정책을 만드는 것이 좋습니다. 이 예제들은 단지 시작점일 뿐이며, 사용자는 이를 활용하여 새롭고 구체적인 정책 배포를 생성할 수 있으며, 심지어 고려하지 않았던 커널 함수까지 추적할 수 있습니다. 어떤 함수를 추적하고 어떤 필터를 적용할지에 대한 구체적인 내용은 엔진 자체에 하드코딩되어 있지 않습니다.
- 중요한 점은 Tetragon을 사용하면 **사용자 공간 애플리케이션이** 데이터 구조를 조작할 수 없는 **커널 깊숙이 후킹**할 수 있으므로 시스템 호출 추적에서 발생하는 일반적인 문제(데이터가 잘못 읽히거나 공격자가 악의적으로 변경하거나 페이지 오류 및 기타 사용자/커널 경계 오류로 인해 데이터가 누락되는 문제)를 피할 수 있다는 것입니다.
- Tetragon 개발자 중 상당수는 커널 개발자이기도 합니다. Tetragon은 이러한 지식 기반을 활용하여 다양한 일반적인 관측 가능성 및 보안 사용 사례를 해결할 수 있는 추적 정책 세트를 개발했습니다.

## **eBPF 커널 인식**

- Tetragon은 eBPF를 통해 **Linux 커널 상태에 접근**할 수 있습니다.
- Tetragon은 이 커널 상태를 Kubernetes 인식 또는 사용자 정책과 결합하여 **커널에서 실시간으로 적용되는 규칙을 생성**할 수 있습니다.
- 이를 통해 프로세스 네임스페이스 및 기능, 소켓을 프로세스에, 프로세스 파일 설명자를 파일 이름에 추가하는 등의 주석을 달고 적용할 수 있습니다.
- 예를 들어, **애플리케이션이 권한을 변경**할 때, 해당 **프로세스가 시스템 호출을 완료**하고 추가 시스템 호출을 실행하기 전에 **경고를 발생**시키거나 **프로세스를 종료하는 정책을 생성**할 수 있습니다.

# Tetragon 설치 - [Docs](https://tetragon.io/docs/getting-started/)
```bash
# /proc 파일 시스템 액세스 필요.
# 기본적으로 Tetragon은 이벤트 로그의 노이즈를 줄이기 위해 kube-system 이벤트를 필터링합니다. 
helm repo add cilium https://helm.cilium.io
helm repo update
helm install tetragon cilium/tetragon -n kube-system
kubectl rollout status -n kube-system ds/tetragon -w

# 확인
k -n kube-system get deploy tetragon-operator -owide
k -n kube-system get cm tetragon-operator-config tetragon-config

k -n kube-system get ds tetragon -owide
k -n kube-system get svc,ep tetragon
k -n kube-system get svc,ep tetragon-operator-metrics

k -n kube-system get pod -l app.kubernetes.io/part-of=tetragon -owide
k -n kube-system get pod -l app.kubernetes.io/name=tetragon
kc -n kube-system describe pod -l app.kubernetes.io/name=tetragon
Containers:
  export-stdout:
    Container ID:  containerd://49be8a9a11ce98c188e0cc964a09454cae39702fad90cc43f5d07e3f6f2e3ba5
    Image:         quay.io/cilium/hubble-export-stdout:v1.1.0

  tetragon:
    Container ID:  containerd://c1956aef6236c2359e4d9905b3c2bceeb1f3b4069ff086eb3c0792c731b3b7bf
    Image:         quay.io/cilium/tetragon:v1.5.0
...
```

## 데모 애플리케이션 배포
```bash
#
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/v1.18.1/examples/minikube/http-sw-app.yaml
service/deathstar created
deployment.apps/deathstar created
pod/tiefighter created
pod/xwing created

# 확인
kubectl get pods

```

# 실행 모니터링 : 시스템의 모든 실행을 추적 - [Docs](https://tetragon.io/docs/getting-started/execution/) , [Blog](https://yuki-nakamura.com/2024/05/23/tetragon-process-lifecycle-observation-tetragon-agent-part/)
![[Pasted image 20250906230138.png]]
![[Pasted image 20250906230152.png]]

## Tetragon은 실행 이벤트를 JSON 로그와 GRPC 스트림을 통해 공개합니다. 사용자는 이를 통해 시스템의 모든 실행 과정을 관찰할 수 있습니다.
```bash
# 여러 노드가 있는 클러스터에서는 사용하는 Tetragon Pod가 "xwing" Pod와 동일한 노드에 있어야 실행 이벤트를 캡처할 수 있습니다.
# 이 명령을 사용하면 "xwing" Pod와 동일한 Kubernetes 노드에 있는 Tetragon Pod의 이름을 가져올 수 있습니다.
POD=$(kubectl -n kube-system get pods -l 'app.kubernetes.io/name=tetragon' -o name --field-selector spec.nodeName=$(kubectl get pod xwing -o jsonpath='{.spec.nodeName}'))
echo $POD
pod/tetragon-62svb

# 터미널1
# 일치하는 Pod를 식별한 후 해당 Pod를 대상으로 지정하여 명령을 실행합니다 : Tetragon에서 캡처한 실행 이벤트를 반환
kubectl exec -ti -n kube-system $POD -c tetragon -- tetra -h
kubectl exec -ti -n kube-system $POD -c tetragon -- tetra getevents -o compact --pods xwing
# 압축 실행 이벤트에는 이벤트 유형, 포드 이름, 바이너리, 인수가 포함됩니다. 종료 이벤트에는 반환 코드가 포함됩니다.
🚀 process default/xwing /usr/bin/bash -c "curl https://ebpf.io/applications/#tetragon" 
🚀 process default/xwing /usr/bin/curl https://ebpf.io/applications/#tetragon 
💥 exit    default/xwing /usr/bin/curl https://ebpf.io/applications/#tetragon 0 

## JSON 형식으로 전체 실행 이벤트를 보려면 명령 -o compact에서 옵션을 제거
kubectl exec -ti -n kube-system $POD -c tetragon -- tetra getevents --pods xwing

# 터미널2
kubectl exec -ti xwing -- bash -c 'curl https://ebpf.io/applications/#tetragon'
kubectl exec -ti xwing -- bash -c 'curl https://httpbin.org'
kubectl exec -ti xwing -- bash -c 'cat /etc/passwd'
```

## (참고) 전체 실행 이벤트
```bash
{
  "process_exec": {
    "process": {
      "exec_id": "Z2tlLWpvaG4tNjMyLWRlZmF1bHQtcG9vbC03MDQxY2FjMC05czk1OjEzNTQ4Njc0MzIxMzczOjUyNjk5",
      "pid": 52699,
      "uid": 0,
      "cwd": "/",
      "binary": "/usr/bin/curl",
      "arguments": "https://ebpf.io/applications/#tetragon",
      "flags": "execve rootcwd",
      "start_time": "2023-10-06T22:03:57.700327580Z",
      "auid": 4294967295,
      "pod": {
        "namespace": "default",
        "name": "xwing",
        "container": {
          "id": "containerd://551e161c47d8ff0eb665438a7bcd5b4e3ef5a297282b40a92b7c77d6bd168eb3",
          "name": "spaceship",
          "image": {
            "id": "docker.io/tgraf/netperf@sha256:8e86f744bfea165fd4ce68caa05abc96500f40130b857773186401926af7e9e6",
            "name": "docker.io/tgraf/netperf:latest"
          },
          "start_time": "2023-10-06T21:52:41Z",
          "pid": 49
        },
        "pod_labels": {
          "app.kubernetes.io/name": "xwing",
          "class": "xwing",
          "org": "alliance"
        },
        "workload": "xwing"
      },
      "docker": "551e161c47d8ff0eb665438a7bcd5b4",
      "parent_exec_id": "Z2tlLWpvaG4tNjMyLWRlZmF1bHQtcG9vbC03MDQxY2FjMC05czk1OjEzNTQ4NjcwODgzMjk5OjUyNjk5",
      "tid": 52699
    },
    "parent": {
      "exec_id": "Z2tlLWpvaG4tNjMyLWRlZmF1bHQtcG9vbC03MDQxY2FjMC05czk1OjEzNTQ4NjcwODgzMjk5OjUyNjk5",
      "pid": 52699,
      "uid": 0,
      "cwd": "/",
      "binary": "/bin/bash",
      "arguments": "-c \"curl https://ebpf.io/applications/#tetragon\"",
      "flags": "execve rootcwd clone",
      "start_time": "2023-10-06T22:03:57.696889812Z",
      "auid": 4294967295,
      "pod": {
        "namespace": "default",
        "name": "xwing",
        "container": {
          "id": "containerd://551e161c47d8ff0eb665438a7bcd5b4e3ef5a297282b40a92b7c77d6bd168eb3",
          "name": "spaceship",
          "image": {
            "id": "docker.io/tgraf/netperf@sha256:8e86f744bfea165fd4ce68caa05abc96500f40130b857773186401926af7e9e6",
            "name": "docker.io/tgraf/netperf:latest"
          },
          "start_time": "2023-10-06T21:52:41Z",
          "pid": 49
        },
        "pod_labels": {
          "app.kubernetes.io/name": "xwing",
          "class": "xwing",
          "org": "alliance"
        },
        "workload": "xwing"
      },
      "docker": "551e161c47d8ff0eb665438a7bcd5b4",
      "parent_exec_id": "Z2tlLWpvaG4tNjMyLWRlZmF1bHQtcG9vbC03MDQxY2FjMC05czk1OjEzNTQ4NjQ1MjQ1ODM5OjUyNjg5",
      "tid": 52699
    }
  },
  "node_name": "gke-john-632-default-pool-7041cac0-9s95",
  "time": "2023-10-06T22:03:57.700326678Z"
}
```


# 파일 접속 모니터링 : 추적 정책으로 민감 파일 모니터링 - [Docs](https://tetragon.io/docs/getting-started/file-events/) , [Blog](https://isovalent.com/blog/post/file-monitoring-with-ebpf-and-tetragon-part-1/)
![[Pasted image 20250906230251.png]]
![[Pasted image 20250906230303.png]]
- YAML 구성 파일을 통해 Tetragon에 **추적 정책**을 추가하면 Tetragon의 기본 실행 추적 기능을 확장할 수 있습니다.
- 이러한 정책은 **커널에서 필터링을 수행**하여 커널에서 실행 중인 BPF 프로그램에서 **관심 있는 이벤트**만 **사용자 공간에 게시**되도록 합니다.
- 이를 통해 사용량이 많은 시스템에서도 **오버헤드를 낮게 유지**할 수 있습니다.
- [아래 지침은 실행 모니터링](https://tetragon.io/docs/getting-started/execution/) 의 예시를 확장하여 Linux에서 **민감한 파일을 모니터링**하는 정책을 적용합니다.
- 사용되는 정책은 이며 [`file_monitoring.yaml`](https://github.com/cilium/tetragon/blob/main/examples/quickstart/file_monitoring.yaml), 필요에 따라 검토하고 확장할 수 있습니다.

## 설정

```bash
# file_monitoring.yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: "file-monitoring-filtered"
spec:
  kprobes:
  - call: "security_file_permission"
    syscall: false
    return: true
    args:
    - index: 0
      type: "file" # (struct file *) used for getting the path
    - index: 1
      type: "int" # 0x04 is MAY_READ, 0x02 is MAY_WRITE
    returnArg:
      index: 0
      type: "int"
    returnArgAction: "Post"
    selectors:
    - matchArgs:      
      - index: 0
        operator: "Prefix"
        values:
        - "/boot"           # Reads to sensitive directories
        - "/root/.ssh"      # Reads to sensitive files we want to know about
        - "/etc/shadow"
        - "/etc/profile"
        - "/etc/sudoers"
        - "/etc/pam.conf"   # Reads global shell configs bash/csh supported
        - "/etc/bashrc"
        - "/etc/csh.cshrc"
        - "/etc/csh.login"  # Add additional sensitive files here
      - index: 1
        operator: "Equal"
        values:
        - "4" # MAY_READ
...
kubectl apply -f https://raw.githubusercontent.com/cilium/tetragon/main/examples/quickstart/file_monitoring.yaml
kubectl get tracingpolicy

```

## Tetragon 파일 액세스 이벤트 관찰
```bash
# 터미널1
POD=$(kubectl -n kube-system get pods -l 'app.kubernetes.io/name=tetragon' -o name --field-selector spec.nodeName=$(kubectl get pod xwing -o jsonpath='{.spec.nodeName}'))
kubectl exec -ti -n kube-system $POD -c tetragon -- tetra getevents -o compact --pods xwing
🚀 process default/xwing /usr/bin/bash -c "cat /etc/shadow"               
🚀 process default/xwing /usr/bin/cat /etc/shadow                         
📚 read    default/xwing /usr/bin/cat /etc/shadow                         
📚 read    default/xwing /usr/bin/cat /etc/shadow                         
📚 read    default/xwing /usr/bin/cat /etc/shadow                         
📚 read    default/xwing /usr/bin/cat /etc/shadow                         
💥 exit    default/xwing /usr/bin/cat /etc/shadow 0  

# 터미널2 : 이벤트를 생성하기위애 정책에 참조된 중요한(민감한) 파일을 읽어보세요
kubectl exec -ti xwing -- bash -c 'cat /etc/shadow'



# 터미널2 : 추적 정책에 따라 Tetragon은 민감한 디렉토리에 쓰기를 시도하는 경우(예: /etc디렉토리에 쓰기를 시도하는 경우)에 대한 응답으로 쓰기 이벤트를 생성합니다.
kubectl exec -ti xwing -- bash -c 'echo foo >> /etc/bar'

# 터미널1
kubectl exec -ti -n kube-system $POD -c tetragon -- tetra getevents -o compact --pods xwing
🚀 process default/xwing /usr/bin/bash -c "echo foo >> /etc/bar"          
📝 write   default/xwing /usr/bin/bash /etc/bar                           
📝 write   default/xwing /usr/bin/bash /etc/bar                           
💥 exit    default/xwing /usr/bin/bash -c "echo foo >> /etc/bar" 0 

# 정책 삭제
kubectl delete -f https://raw.githubusercontent.com/cilium/tetragon/main/examples/quickstart/file_monitoring.yaml

```



# Policy Enforcement : 커널 수준에서 정책 제한을 적용 - [Docs](https://tetragon.io/docs/getting-started/enforcement/)
- Tetragon의 추적 정책은 파일 액세스 이벤트나 네트워크 연결 이벤트와 같은 이벤트를 보고하기 위해 커널 함수를 모니터링하고, 해당 커널 함수에 제한을 적용할 수 있도록 지원합니다.
- Tetragon에서 커널 내 필터링을 사용하면 커널에서 사용자 공간으로 전송되는 이벤트를 제한하여 성능을 크게 향상시킬 수 있습니다.
- 또한, 커널 내 필터링을 통해 Tetragon은 커널 수준에서 정책 제한을 적용할 수 있습니다.
- 예를 들어, `SIGKILL`정책 위반이 감지되었을 때 프로세스에 를 실행하면 해당 프로세스는 더 이상 실행되지 않습니다.
- 정책 적용이 시스템 호출을 통해 트리거되는 경우, 애플리케이션은 시스템 호출에서 반환되지 않고 종료됩니다.
- 이 섹션에서는 이 시작 가이드에서 이미 배포한 Tetragon 기능(실행, 파일 추적 및 네트워크 추적 정책)에 네트워크 및 파일 정책 적용을 추가합니다.
- 구체적으로는 다음과 같습니다.
    - Kubernetes 클러스터에서 나가는 **네트워크 트래픽을 제한**하는 정책 적용
    - **민감한 파일에 블록 쓰기 및 읽기 작업** 적용
- 구체적인 구현 세부 사항은 [시행](https://tetragon.io/docs/concepts/enforcement/) 개념 섹션을 참조하세요.

## **파일 액세스 제한 적용**

- [다음은 파일 액세스 모니터링](https://tetragon.io/docs/getting-started/file-events/) 의 예시를 적용하여 민감한 파일이 읽히지 않도록 확장하는 방법입니다.
- 사용되는 정책은 이며 [`file_monitoring_enforce.yaml`](https://github.com/cilium/tetragon/blob/main/examples/quickstart/file_monitoring_enforce.yaml), 필요에 따라 검토하고 확장할 수 있습니다.
- 관찰 정책과 적용 정책의 유일한 차이점은 애플리케이션에 작업 블록을 추가 `SIGKILL`하고 작업 시 오류를 반환한다는 것입니다.
```bash
# file_monitoring_enforce.yaml : matchaction 확인
apiVersion: cilium.io/v1alpha1
kind: TracingPolicyNamespaced
metadata:
  name: "file-monitoring-filtered"
spec:
  kprobes:
  - call: "security_file_permission"
    syscall: false
    return: true
    args:
    - index: 0
      type: "file" # (struct file *) used for getting the path
    - index: 1
      type: "int" # 0x04 is MAY_READ, 0x02 is MAY_WRITE
    returnArg:
      index: 0
      type: "int"
    returnArgAction: "Post"
    selectors:
    - matchArgs:
      - index: 0
        operator: "Prefix"
        values:
        - "/boot"           # Reads to sensitive directories
        - "/root/.ssh"      # Reads to sensitive files we want to know about
        - "/etc/shadow"
        - "/etc/profile"
        - "/etc/sudoers"
        - "/etc/pam.conf"   # Reads global shell configs bash/csh supported
        - "/etc/bashrc"
        - "/etc/csh.cshrc"
        - "/etc/csh.login"  # Add additional sensitive files here
      - index: 1
        operator: "Equal"
        values:
        - "4" # MAY_READ
      matchActions:
      - action: Sigkill
...
kubectl apply -f https://raw.githubusercontent.com/cilium/tetragon/main/examples/quickstart/file_monitoring_enforce.yaml
kubectl get tracingpolicynamespaced


# 터미널1
POD=$(kubectl -n kube-system get pods -l 'app.kubernetes.io/name=tetragon' -o name --field-selector spec.nodeName=$(kubectl get pod xwing -o jsonpath='{.spec.nodeName}'))
kubectl exec -ti -n kube-system $POD -c tetragon -- tetra getevents -o compact --pods xwing
🚀 process default/xwing /usr/bin/bash -c "cat /etc/shadow"               
🚀 process default/xwing /usr/bin/cat /etc/shadow                         
📚 read    default/xwing /usr/bin/cat /etc/shadow                         
📚 read    default/xwing /usr/bin/cat /etc/shadow                         
💥 exit    default/xwing /usr/bin/cat /etc/shadow SIGKILL

# 터미널2 : 정의된 정책에 포함된 파일 중 하나인 민감한 파일을 읽어보세요
kubectl exec -ti xwing -- bash -c 'cat /etc/shadow'
command terminated with exit code 137

# 시행된 파일 정책에 포함되지 않은 파일을 읽거나 쓰려는 시도는 영향을 받지 않습니다.
kubectl exec -ti xwing -- bash -c 'echo foo > /tmp/test.txt'
```

# (정보) Use Cases : Part 1 - [Blog](https://isovalent.com/blog/post/top-tetragon-use-cases/) & Part 2 - [Blog](https://isovalent.com/blog/post/top-tetragon-use-cases-part-2/)

# (정보) eBPF and Tetragon: Tools for Detecting XZ Utils CVE 2024-3094 Exploit - [Blog](https://isovalent.com/blog/post/ebpf-tetragon-xz-utils-cve-policy)
```bash
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: "cve-2024-3094-xz-ssh"
  annotations:
    url: "https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-3094"
    description: "Detects if OpenSSH is using vulnerable XZ libraries"
    author: "Tetragon.io Team"
spec:
  kprobes:
  - call: "security_mmap_file"
    syscall: false
    return: true
    # message: "OpenSSH daemon using vulnerable XZ libraries CVE-2024-3094"
    # tags: [ "cve", "cve.2024.3094" ]
    args:
    - index: 0
      type: "file"
    - index: 1
      type: "uint32"
    - index: 2
      type: "nop"
    returnArg:
      index: 0
      type: "int"
    returnArgAction: "Post"
    selectors:
    - matchBinaries:
      - operator: "In"
        values:
        - "/usr/sbin/sshd"
      matchArgs:
      - index: 0
        operator: "Postfix"
        values:
        - "liblzma.so.5.6.0"
        - "liblzma.so.5.6.1"
      matchActions:
        - action: Post
          rateLimit: "1m"
```

# (정보) Building a Professional-Grade DevSecOps Pipeline with Tetragon eBPF Security Monitoring on Azure AKS : 보안 정책 적용/감사 파이프라인 - [Blog](https://medium.com/careerbytecode/building-a-professional-grade-devsecops-pipeline-with-tetragon-ebpf-security-monitoring-on-azure-6a36b863ee37)

## 전체 파일 구조
```bash
tetragon-devsecops/ 
├── app.js                               # 데모 애플리케이션
 ├── Dockerfile                           # 보안 컨테이너 빌드
 ├── package.json                         # 종속성
 ├── azure-pipelines.yml                  # 전체 CI/CD 파이프라인
 ├── k8s-manifests/ 
│ ├── deployment.yaml                  # 보안 Pod 구성
 │ └── service.yaml                     # 로드 밸런서 구성
 └── security/ 
    ├── tetragon-namespace-policies.yaml # 런타임 보안 정책
     ├── tetragon-policies.yaml           # 클러스터 전체 정책
     ├── tetragon-rbac.yaml               # RBAC 구성
     └── kubescape-config.yaml            # 규정 준수 스캐닝 구성
```

## File Access Monitoring Policy
```bash
apiVersion: cilium.io/v1alpha1
kind: TracingPolicyNamespaced
metadata:
  name: demo-app-file-access
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: tetragon-demo-app
  kprobes:
  - call: "sys_openat"
    syscall: true
    selectors:
    - matchArgs:
      - index: 1
        operator: "Prefix"
        values:
        - "/etc/"
        - "/var/"
        - "/tmp/"
```

## Process Execution Monitoring Policy
```bash
apiVersion: cilium.io/v1alpha1
kind: TracingPolicyNamespaced
metadata:
  name: demo-app-process-execution
spec:
  kprobes:
  - call: "sys_execve"
    syscall: true
    selectors:
    - matchArgs:
      - index: 0
        operator: "Prefix"
        values:
        - "/bin/"
        - "/usr/bin/"
        - "/sbin/"
```

## CI/CD Pileline
![[Pasted image 20250906230516.png]]



# 도전과제10 Making Damn Vulnerable Web Application (DVWA) almost unhackable with Cilium and Tetragon : Tetragon 을 이용하여 DVWA 공격 차단해보기 - [Blog](https://holdmybeersecurity.com/2024/07/24/making-damn-vulnerable-web-application-dvwa-almost-unhackable-with-cilium-and-tetragon/)

^f2b08a

```bash
# Example
---
apiVersion: cilium.io/v1alpha1
kind: TracingPolicyNamespaced
metadata:
  name: "command-line-injection"
  namespace: "dvwa"
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/instance: dvwa
      app.kubernetes.io/name: dvwa
  kprobes:
  - call: "sys_execve"
    syscall: true
    return: true
    args:
    - index: 0
      type: "string" # file path
    returnArg:
      index: 0
      type: "int"
    returnArgAction: "Post"
    selectors:
    - matchPIDs:
      - operator: In
        followForks: true
        isNamespacePID: true
        values:
          - 1 # Apache root 
      matchArgs:      
      - index: 0
        operator: "NotEqual"
        values:
          - "/usr/bin/ping"
          - "/bin/sh"
      matchActions:
      - action: Override
        argError: -1
      - action: Post
```