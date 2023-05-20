### Lab3 deploy workload with rancher apps.md


**1) 클러스터 Apps을 이용한 서비스 설치**

- local-path storeage class 설치
- tomcat apps 설치


```bash
# loca-path storage 설치
$ kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.22/deploy/local-path-storage.yaml
$ kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

- Cluster > Apps > Charts > Filter tomcat 선택
- Apache Tomcat > Install > Namespace > tomcat 입력 > Next
- service: type: LoadBalancer 값을 type: ClusterIP로 변경 > Install

- Cluster > Services > tomcat > http 클릭


**2) nginx 웹 서버 설치**

- Cluster > local > Worksloads > Create
- Deployment > Namespace > Create a New Namespace : nginx 입력 
- Name : nginx > Container Image : nginx 
- Networking > ClusterIP > Name : http > Private Container Port : 80 > Create


- Cluster > Services > nginx > http 클릭
