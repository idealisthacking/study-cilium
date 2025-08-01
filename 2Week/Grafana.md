- 그라파나 Panel 패널 - [Link](https://grafana.com/docs/grafana/latest/panels-visualizations/)
	- [https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/)
    - Graphs & charts  
        - [Time series](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/time-series/) is the default and main Graph visualization.
        - [State timeline](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/state-timeline/) for state changes over time.
        - [Status history](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/status-history/) for periodic state over time.
        - [Bar chart](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/bar-chart/) shows any categorical data.
        - [Histogram](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/histogram/) calculates and shows value distribution in a bar chart.
        - [Heatmap](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/heatmap/) visualizes data in two dimensions, used typically for the magnitude of a phenomenon.
        - [Pie chart](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/pie-chart/) is typically used where proportionality is important.
        - [Candlestick](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/candlestick/) is typically for financial data where the focus is price/data movement.
        - [Gauge](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/gauge/) is the traditional rounded visual showing how far a single metric is from a threshold.
    - Stats & numbers
        - [Stat](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/stat/) for big stats and optional sparkline.
        - [Bar gauge](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/bar-gauge/) is a horizontal or vertical bar gauge.
    - Misc
        - [Table](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/table/) is the main and only table visualization.
        - [Logs](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/logs/) is the main visualization for logs.
        - [Node graph](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/node-graph/) for directed graphs or networks.
        - [Traces](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/traces/) is the main visualization for traces.
        - [Flame graph](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/flame-graph/) is the main visualization for profiling.
        - [Canvas](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/canvas/) allows you to explicitly place elements within static and dynamic layouts.
        - [Geomap](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/geomap/) helps you visualize geospatial data.
    - Widgets
        - [Dashboard list](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/dashboard-list/) can list dashboards.
        - [Alert list](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/alert-list/) can list alerts.
        - [Text](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/text/) can show markdown and html.
        - [News](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/news/) can show RSS feeds.
   
	- 실습 준비 : 신규 대시보스 생성 → 패널 생성(Code 로 변경) → 쿼리 입력 후 Run queries 클릭 후 오른쪽 상단 Apply 클릭 → 대시보드 상단 저장
        

1. [Time series](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/time-series/) : 아래 쿼리 입력 후 오른쪽 입력 → Title(노드별 5분간 CPU 사용 변화율)
	
```bash
node_cpu_seconds_total
rate(node_cpu_seconds_total[5m])
sum(rate(node_cpu_seconds_total[5m]))
sum(rate(node_cpu_seconds_total[5m])) by (instance)
```
	
- 상단 : 최근 30분, 5초 간격 Auto 쿼리
2. [Bar chart](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/bar-chart/) : Add → Visualization 오른쪽(Bar chart) ⇒ 쿼리 Options : Legend(Auto), Format(Table), Type(Instance) → Title(네임스페이스 별 디플로이먼트 갯수)

```bash
kube_deployment_status_replicas_available
count(kube_deployment_status_replicas_available) by (namespace)
```
	
3. [Stat](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/stat/) : Add → Visualization 오른쪽(Stat) → Title(nginx 파드 수)

```bash
kube_deployment_spec_replicas
kube_deployment_spec_replicas{deployment="nginx"}

# scale out
kubectl scale deployment nginx --replicas 6
```
	
4. [Gauge](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/gauge/) : Add → Visualization 오른쪽(Gauge) → Title(노드 별 1분간 CPU 사용률)

```bash
node_cpu_seconds_total
node_cpu_seconds_total{mode="idle"}
node_cpu_seconds_total{mode="idle"}[1m]
rate(node_cpu_seconds_total{mode="idle"}[1m])
avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) by (instance)
1 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) by (instance))
```
	
5. [Table](https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/table/) : Add → Visualization 오른쪽(Table) ⇒ 쿼리 Options : Format(Table), Type(Instance) → Title(노드 OS 정보)

```bash
node_os_info
```
- Transform data → Organize fields by name : id_like, instance, name, pretty_name
- 
6. 원하는 위치로 배치
	![[Pasted image 20250727013833.png]]

7. [도전] 대시보드에 Variables 에 instance 를 설정 후 필터링 출력 되게 해보자!
![[Pasted image 20250727014037.png]]
![[Pasted image 20250727014001.png]]


- 그라파나 얼럿 Docs - [링크](https://grafana.com/docs/grafana/latest/alerting/) [Workshop](https://catalog.workshops.aws/observability/en-US/aws-managed-oss/amg/alerts-notifications) [Blog1](https://velog.io/@yyw1128/4) [Blog2](https://blog.naver.com/sungwon_200ok/223061416989)
![[Pasted image 20250727014109.png]]
[https://grafana.com/docs/grafana/latest/alerting/](https://grafana.com/docs/grafana/latest/alerting/)

1. Contact points → Add contact point 클릭
- Integration : 슬랙
- Webhook URL : 아래 주소 입력
		
```bash
<https://hooks.slack.com/services/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>
```
- Optional Slack settings → Username : 메시지 구분을 위해서 각자 자신의 닉네임 입력
- 오른쪽 상단 : Test 해보고 저장
![[Pasted image 20250727014233.png]]

2. Notification policies : 기본 정책 수정 Edit - Default contact point(slack)
![[Pasted image 20250727014244.png]]
	
3. 그라파나 → Alerting → Alert ruels → Create alert rule : Name(nginx alert) - nginx 웹 요청 1분 동안 누적 60 이상 시 Alert 설정
![[Pasted image 20250727014309.png]]

- 아래 Folder 과 Evaluation group(1m), Pending period(1m) 은 +Add new 클릭 후 신규로 만들어 주자
- Configure notifications : Contack point(slack)
	⇒ 오른쪽 상단 `Save and exit` 클릭
	
4.nginx 반복 접속 실행 후 슬랙 채널 알람 확인
```bash
while true; do curl -s <https://nginx.$MyDomain> -I | head -n 1; date; done
```
![[Pasted image 20250727014401.png]]
