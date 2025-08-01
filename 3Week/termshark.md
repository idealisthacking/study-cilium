Termshark - [Github](https://github.com/gcla/termshark) , [Home](https://termshark.io/)

```bash
# 이미 설치되어 있음
export DEBIAN_FRONTEND=noninteractive
apt-get install -y termshark

# Inspect a local pcap:
termshark -r test.pcap

# Capture ping packets on interface eth0:
termshark -i eth0 icmp
```