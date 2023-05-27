### A1. Spring Monitoring with Prometheus Service Monitor and Grafana

---

**1) Add Service Monitor**
~~~
$ k apply -f grafana/spring-monitor.yml
~~~

- Cluster > Local > Monitoring > Prometheus Targets 에 등록된 서비스 모니터를 확인합니다.

---

**2) Add Grafana Dashboard**

- Cluster > Local > Monitoring > Grafana에 접속합니다.
- Login (admin / prom-operator)로 로그인 합니다.
- Import > Import via panel json
- Copy and Paste grafana/spring-monitor.json

---

**3) Check Dashboard**
