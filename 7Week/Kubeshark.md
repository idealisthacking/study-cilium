# Inspect all internal and external cluster communications, API calls, and data in transit. - [Home](https://www.kubeshark.co/) , [Docs](https://docs.kubeshark.co/en/introduction) , [Blog](https://www.kubeshark.co/blog) , [demo](https://demo.kubeshark.co/)

![[Pasted image 20250830002739.png]]

![[Pasted image 20250830002750.png]]

# 설치 - [Docs](https://docs.kubeshark.co/en/install)

```bash
# mac
brew install kubeshark

# 확인
kubeshark version
v52.8.1

kubeshark -h
Available Commands:
  clean       Removes all Kubeshark resources
  completion  Generate the autocompletion script for the specified shell
  config      Generate Kubeshark config with default values
  console     Stream the scripting console logs into shell
  help        Help about any command
  license     Print the license loaded string
  logs        Create a ZIP file with logs for GitHub issues or troubleshooting
  pcapdump    Store all captured traffic (including decrypted TLS) in a PCAP file.
  pprof       Select a Kubeshark container and open the pprof web UI in the browser
  proxy       Open the web UI (front-end) in the browser via proxy/port-forward
  scripts     Watch the `scripting.source` and/or `scripting.sources` folders for changes and update the scripts
  tap         Capture the network traffic in your Kubernetes cluster
  version     Print version info
 
# Capture the network traffic in your Kubernetes cluster
kubeshark tap
2025-08-27T23:25:47+09:00 INF tapRunner.go:47 > Using Docker: registry=docker.io/kubeshark tag=
2025-08-27T23:25:47+09:00 INF tapRunner.go:51 > Kubeshark will store the traffic up to a limit (per node). Oldest TCP/UDP streams will be removed once the limit is reached. limit=5Gi
2025-08-27T23:25:47+09:00 INF versionCheck.go:23 > Checking for a newer version...
2025-08-27T23:25:47+09:00 INF common.go:69 > Using kubeconfig: path=/Users/gasida/.kube/config
2025-08-27T23:25:48+09:00 INF tapRunner.go:67 > Telemetry enabled=true notice="Telemetry can be disabled by setting the flag: --telemetry-enabled=false"
2025-08-27T23:25:48+09:00 INF tapRunner.go:69 > Targeting pods in: namespaces=["cilium-secrets","default","kube-node-lease","kube-public","kube-system","local-path-storage","monitoring"]
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: cilium-envoy-xk459
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: cilium-operator-74c8d8bb68-5r657
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: cilium-t2mgt
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: coredns-674b8bbfcf-4l4sr
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: coredns-674b8bbfcf-877hc
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: etcd-myk8s-control-plane
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: hubble-relay-fdd49b976-xjzpn
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: hubble-ui-655f947f96-mc2tf
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: kube-apiserver-myk8s-control-plane
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: kube-controller-manager-myk8s-control-plane
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: kube-ops-view-6658c477d4-dcnzp
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: kube-scheduler-myk8s-control-plane
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: local-path-provisioner-7dc846544d-jhv4n
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: kube-prometheus-stack-grafana-0
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: kube-prometheus-stack-kube-state-metrics-85f6b56f69-h4pj7
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: kube-prometheus-stack-operator-9cbf49c6c-96qs8
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: kube-prometheus-stack-prometheus-node-exporter-7m527
2025-08-27T23:25:48+09:00 INF tapRunner.go:136 > Targeted pod: prometheus-kube-prometheus-stack-prometheus-0
2025-08-27T23:25:48+09:00 INF tapRunner.go:79 > Waiting for the creation of Kubeshark resources...
2025-08-27T23:25:52+09:00 INF helm.go:128 > Downloading Helm chart: repo-path=/Users/gasida/Library/Caches/helm/repository url=https://github.com/kubeshark/kubeshark.github.io/releases/download/kubeshark-52.8.1/kubeshark-52.8.1.tgz
2025-08-27T23:25:54+09:00 INF helm.go:147 > Installing using Helm: kube-version=">= 1.16.0-0" release=kubeshark source=["https://github.com/kubeshark/kubeshark/tree/master/helm-chart"] version=52.8.1
2025-08-27T23:25:55+09:00 INF helm.go:61 > creating 21 resource(s)
2025-08-27T23:25:55+09:00 INF tapRunner.go:96 > Installed the Helm release: kubeshark
2025-08-27T23:25:55+09:00 INF tapRunner.go:167 > Added: pod=kubeshark-hub
2025-08-27T23:25:55+09:00 INF tapRunner.go:258 > Added: pod=kubeshark-front
2025-08-27T23:26:20+09:00 INF tapRunner.go:192 > Ready. pod=kubeshark-hub
2025-08-27T23:26:31+09:00 INF tapRunner.go:282 > Ready. pod=kubeshark-front
2025-08-27T23:26:31+09:00 INF proxy.go:31 > Starting proxy... namespace=default proxy-host=127.0.0.1 service=kubeshark-front src-port=8899
2025-08-27T23:26:36+09:00 INF tapRunner.go:417 > Kubeshark is available at: url=http://127.0.0.1:8899
2025-08-27T23:26:36+09:00 INF console.go:45 > Starting scripting console ...
E0827 23:26:38.191099   26740 proxy_server.go:147] Error while proxying request: error dialing backend: context canceled
2025-08-27T23:26:41+09:00 INF console.go:61 > Connecting to: host=127.0.0.1 url=http://127.0.0.1:8899/api
```

![[Pasted image 20250830002812.png]]
![[Pasted image 20250830002821.png]]

# (옵션) 폐쇄망 Air-Gapped 환경 : Enterprise Tier(Plan)에서만 가능 - [Docs](https://www.kubeshark.co/pricing)
```bash
# Make sure to disable internet connectivity:
internetConnectivity: false
```

## (참고) 삭제 시

```bash
# cleanup  
kubeshark clean  
```