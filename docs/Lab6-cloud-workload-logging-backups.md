### Cloud Workload Loggign and Backups

---
<img src="./logging.png" width="850" height="700">

**1. Logging Operator 설정**

- Logging Operator 설치
- Rancher 로그인 > 클러스터 이동
- Cluster Tools > Logging > Install > Install into Project > Observability > Next > Install # 기본값 설치
- cattle-logging-system 내 fluentd / fluentbit 생성 확인

- loki 설치

```bash
$ helm install loki loki/loki -n loki --create-namespace
```

- Logging / Flow / Output 생성 (loki)

```bash
# logging 생성  
$ kubectl -n cattle-logging-system apply -f - <<"EOF"
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: kw01-logging
spec:
  fluentd:
    logLevel: debug
  fluentbit: {}
  controlNamespace: cattle-logging-system
EOF
  
# loki output 생성  
$ kubectl -n deploy-kust-dev apply -f - <<"EOF"
apiVersion: logging.banzaicloud.io/v1beta1
kind: Output
metadata:
 name: loki-output
spec:
 loki:
   url: http://loki.loki:3000
   configure_kubernetes_labels: true
   buffer:
     timekey: 1m
     timekey_wait: 30s
     timekey_use_utc: true
EOF

#Log Flow 생성

$ kubectl -n deploy-kust-dev apply -f - <<"EOF"
apiVersion: logging.banzaicloud.io/v1beta1
kind: Flow
metadata:
  name: kw01-flow
  namespace: deploy-kust-dev
spec:
  filters:
    - tag_normaliser: {}
    - {}
  localOutputRefs:
    - loki-output
  match:
    - select:
        labels:
          stage: dev
EOF
 
$ k get flow -n deploy-kust-dev
$ k get ouput -n deploy-kust-dev
 

#Nginx Logger 설치

$ kubectl -n deploy-kust-dev apply -f - <<"EOF"
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-logger
    stage: dev
  name: nginx-logger
  namespace: deploy-kust-dev
spec:
  selector:
    matchLabels:
      app: nginx-logger
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx-logger
        stage: dev
    spec:
      containers:
        - image: kscarlett/nginx-log-generator
          name: nginx-log-generator
EOF
``` 

- Loki 로그 
- Loki (admin / prom-operator)
  https://rancher.kw01/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy/?orgId=1
