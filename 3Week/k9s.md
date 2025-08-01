Kubernetes CLI To Manage Your Clusters In Style! - [Github](https://github.com/derailed/k9s)

# 설치
```bash
# arm64 CPU
wget https://github.com/derailed/k9s/releases/latest/download/k9s_linux_arm64.deb -O /tmp/k9s_linux_arm64.deb
apt install /tmp/k9s_linux_arm64.deb

# amd64 CPU
wget https://github.com/derailed/k9s/releases/latest/download/k9s_linux_amd64.deb -O /tmp/k9s_linux_amd64.deb
apt install /tmp/k9s_linux_amd64.deb

#
which k9s
```
# The Command Line [https://peterica.tistory.com/276](https://peterica.tistory.com/276)

```bash
# List current version
k9s version

# To get info about K9s runtime (logs, configs, etc..)
k9s info

# List all available CLI options
k9s help

# To run K9s in a given namespace
k9s -n mycoolns

# Start K9s in an existing KubeConfig context
k9s --context coolCtx

# Start K9s in readonly mode - with all cluster modification commands disabled
k9s --readonly
```

# Key Bindings
|**Action**|**Command**|**Comment**|
|---|---|---|
|Show active keyboard mnemonics and help|`?`||
|Show all available resource alias|`ctrl-a`||
|To bail out of K9s|`:quit`, `:q`, `ctrl-c`||
|To go up/back to the previous view|`esc`|If you have crumbs on, this will go to the previous one|
|View a Kubernetes resource using singular/plural or short-name|`:`pod⏎|accepts singular, plural, short-name or alias ie pod or pods|
|View a Kubernetes resource in a given namespace|`:`pod ns-x⏎||
|View filtered pods (New v0.30.0!)|`:`pod /fred⏎|View all pods filtered by fred|
|View labeled pods (New v0.30.0!)|`:`pod app=fred,env=dev⏎|View all pods with labels matching app=fred and env=dev|
|View pods in a given context (New v0.30.0!)|`:`pod @ctx1⏎|View all pods in context ctx1. Switches out your current k9s context!|
|Filter out a resource view given a filter|`/`filter⏎|Regex2 supported ie `fred|
|Inverse regex filter|`/`! filter⏎|Keep everything that _doesn't_ match.|
|Filter resource view by labels|`/`-l label-selector⏎||
|Fuzzy find a resource given a filter|`/`-f filter⏎||
|Bails out of view/command/filter mode|`<esc>`||
|Key mapping to describe, view, edit, view logs,...|`d`,`v`, `e`, `l`,...||
|To view and switch to another Kubernetes context (Pod view)|`:`ctx⏎||
|To view and switch directly to another Kubernetes context (Last used view)|`:`ctx context-name⏎||
|To view and switch to another Kubernetes namespace|`:`ns⏎||
|To switch back to the last active command (like how "cd -" works)|`-`|Navigation that adds breadcrumbs to the bottom are not commands|
|To go back and forward through the command history|back: `[`, forward: `]`|Same as above|
|To view all saved resources|`:`screendump or sd⏎||
|To delete a resource (TAB and ENTER to confirm)|`ctrl-d`||
|To kill a resource (no confirmation dialog, equivalent to kubectl delete --now)|`ctrl-k`||
|Launch pulses view|`:`**pulses** or pu⏎||
|Launch XRay view|`:`**xray** RESOURCE [NAMESPACE]⏎|RESOURCE can be one of po, **svc**, dp, rs, sts, ds, NAMESPACE is optional|
|Launch Popeye view|`:`popeye or pop⏎|See [popeye](https://github.com/derailed/k9s#popeye)|

