```bash
# cilium 파드 이름
export CILIUMPOD0=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-ctr -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD1=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w1  -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD2=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w2  -o jsonpath='{.items[0].metadata.name}')
echo $CILIUMPOD0 $CILIUMPOD1 $CILIUMPOD2

# 단축키(alias) 지정
alias c0="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- cilium"
alias c1="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- cilium"
alias c2="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- cilium"

alias c0bpf="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- bpftool"
alias c1bpf="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- bpftool"
alias c2bpf="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- bpftool"


# endpoint
c0 endpoint list
c0 endpoint list -o json
c1 endpoint list
c2 endpoint list

c1 endpoint get <id>
c1 endpoint log <id>

## Enable debugging output on the cilium-dbg monitor for this endpoint
c1 endpoint config <id> Debug=true


# monitor
c1 monitor
c1 monitor -v
c1 monitor -v -v

## Filter for only the events related to endpoint
c1 monitor --related-to=<id>

## Show notifications only for dropped packet events
c1 monitor --type drop

## Don’t dissect packet payload, display payload in hex information
c1 monitor -v -v --hex

## Layer7
c1 monitor -v --type l7


# Manage IP addresses and associated information - IP List
c0 ip list

# IDENTITY :  1(host), 2(world), 4(health), 6(remote), 파드마다 개별 ID
c0 ip list -n

# Retrieve information about an identity
c0 identity list

# 엔드포인트 기준 ID
c0 identity list --endpoints

# 엔드포인트 설정 확인 및 변경
c0 endpoint config <엔트포인트ID>

# 엔드포인트 상세 정보 확인
c0 endpoint get <엔트포인트ID>

# 엔드포인트 로그 확인
c0 endpoint log <엔트포인트ID>

# Show bpf filesystem mount details
c0 bpf fs show

# bfp 마운트 폴더 확인
tree /sys/fs/bpf


# Get list of loadbalancer services
c0 service list
c1 service list
c2 service list

## Or you can get the loadbalancer information using bpf list
c0 bpf lb list
c1 bpf lb list
c2 bpf lb list

## List reverse NAT entries
c1 bpf lb list --revnat
c2 bpf lb list --revnat


# List connection tracking entries
c0 bpf ct list global
c1 bpf ct list global
c2 bpf ct list global

# Flush connection tracking entries
c0 bpf ct flush
c1 bpf ct flush
c2 bpf ct flush


# List all NAT mapping entries
c0 bpf nat list
c1 bpf nat list
c2 bpf nat list

# Flush all NAT mapping entries
c0 bpf nat flush
c1 bpf nat flush
c2 bpf nat flush

# Manage the IPCache mappings for IP/CIDR <-> Identity
c0 bpf ipcache list# Display cgroup metadata maintained by Cilium
c0 cgroups list
c1 cgroups list
c2 cgroups list


# List all open BPF maps
c0 map list
c1 map list --verbose
c2 map list --verbose

c1 map events cilium_lb4_services_v2
c1 map events cilium_lb4_reverse_nat
c1 map events cilium_lxc
c1 map events cilium_ipcache


# List all metrics
c1 metrics list


# List contents of a policy BPF map : Dump all policy maps
c0 bpf policy get --all
c1 bpf policy get --all -n
c2 bpf policy get --all -n


# Dump StateDB contents as JSON
c0 statedb dump


#
c0 shell -- db/show devices
c1 shell -- db/show devices
c2 shell -- db/show devices

```