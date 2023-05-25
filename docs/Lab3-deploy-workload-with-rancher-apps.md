### Lab 3. deploy workload with rancher apps.md

- Rancher의 Apps 패키지를 이용하여 클러스터 서비스를 설치합니다.
- Helm chart 레파지토리를 추가하여 관리할 수 있습니다.

---

**1) 클러스터 Apps을 이용한 서비스 설치**

- local-path storeage class를 설치합니다.
- tomcat apps 설치하고 Rancher Proxy로 접속합니다.


```bash
# loca-path storage 설치
$ kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.22/deploy/local-path-storage.yaml
$ kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

- Cluster > Apps > Charts > Filter tomcat 선택합니다.
- Apache Tomcat > Install > Namespace > tomcat 입력 > Next 를 선택합니다.
- service: type: LoadBalancer 값을 type: ClusterIP로 변경 > Install 을 선택합니다.

- Cluster > Services > tomcat > http 클릭합니다

---

**2) nginx 웹 서버 설치**

- Cluster > local > Worksloads > Create
- Deployment > Namespace > Create a New Namespace : nginx 입력합니다.
- Name : nginx > Container Image : nginx 를 입력합니다.
- Networking > ClusterIP > Name : http > Private Container Port : 80 > Create 를 선택합니다.


- Cluster > Services > nginx > http 클릭합니다. 
