# Cilium 에서 추천하는 tuned network-*profiles - [Docs](https://docs.cilium.io/en/stable/operations/performance/tuning/#tuned-network-profiles)
- [https://github.com/redhat-performance/tuned](https://github.com/redhat-performance/tuned)
- [https://github.com/redhat-performance/tuned/releases](https://github.com/redhat-performance/tuned/releases)
```bash
# 설치
dnf install tuned

# 서비스 시작
systemctl start tuned
systemctl enable tuned

# To see the current active profile, run:
tuned-adm active

# To list all available profiles, run:
tuned-adm list

# To switch to a different profile, run:
## The enabled profile is persisted into /etc/tuned/active_profile, which is read when the daemon starts or is restarted.
tuned-adm profile <profile-name>

# To disable all tunings, run:
tuned-adm off
...  

# The tuned project offers various profiles to optimize for deterministic performance at the cost of increased power consumption, 
# that is, network-latency and network-throughput, for example. To enable the former, run:
tuned-adm profile network-latency
or 
tuned-adm profile network-throughput

```

[https://tuned-project.org/docs/tuned_devconf_2019.pdf](https://tuned-project.org/docs/tuned_devconf_2019.pdf)

**TuneD** is a system tuning service for Linux. It: - [Home](https://tuned-project.org/) , [Github](https://github.com/redhat-performance/tuned)
- monitors connected devices using the `udev` device manager
- tunes system settings according to a selected profile
- supports various types of configuration like `sysctl`, `sysfs`, or kernel boot command line parameters, which are integrated in a plug-in architecture
- supports hot plugging of devices and can be controlled from the command line or through D-Bus, so it can be easily integrated into existing administering solutions: for example, with Cockpit
- can be run in no-daemon mode with limited functionality (for example, no support for D-Bus, `udev`, tuning of newly created processes, and so on) for systems with reduced resources
- stores all its configuration cleanly in one place – in the TuneD profile – instead of having configuration on multiple places and in custom scripts

**TuneD profiles**:
- can be defined hierarchically, which reduces duplication and simplifies maintenance:
    More specialized profiles can inherit generic profiles and just change what is needed instead of duplicating the code. For example, you can built a generic profile for HTTP server upon the `throughput-performance` profile and later create two more specialized profiles for the Apache server and the Nginx server, basing both profiles on the generic HTTP server profile.
- support full rollback:
    The system can be easily returned to the state before the profile was applied. This can be handy for testing, benchmarking, experimenting, and so on. For example, you can set up a `cron` rule to apply a certain profile during business hours and a different one at night.
- include a number of predefined profiles for common use cases:
    For example, presets for high throughput, low latency, or powersave are distributed. Profiles optimizing performance for various products like SAP, dBase servers, and so on are also provided, and it is possible to fully customize them.

