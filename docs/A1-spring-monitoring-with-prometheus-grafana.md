### A1/ Spring Monitoring with Prometheus Service Monitor and Grafana

---

**1) Add Service Monitor**
~~~
$ k apply -f grafana/spring-monitor.yml
~~~

- Cluster > rke > Project > APPS > Resources > Istio > Prometheus Link
- Status > Targets

---

**2) Add Grafana Dashboard**

- Cluster > rke > Cluster-Metrics > Grafana Link
- Login admin / admin
- Import > Import via panel json
- Copy and Paste grafana/spring-monitor.json

---

**3) Check Dashboard**
