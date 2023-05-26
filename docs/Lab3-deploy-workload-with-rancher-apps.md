### Lab 3. deploy workload with rancher apps.md

- Rancher의 Apps 패키지를 이용하여 클러스터 서비스를 설치합니다.
- Helm chart 레파지토리를 추가하여 관리할 수 있습니다.

---

**1) 클러스터 Apps을 이용한 서비스 설치**

- local-path storeage class를 설치합니다.
- tomcat apps 설치하고 Rancher Proxy로 접속합니다.


```bash
# local-path storage 설치
$ kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.24/deploy/local-path-storage.yaml
$ kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# rhel selinux를 사용하는 경우 local-path-storage 사용에 제약이 있어 disbled 합니다.
$ sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
$ reboot
```

- Rancher에 로그인 한 후
- Cluster > Apps > Charts > Filter tomcat 선택합니다.
- Apache Tomcat > Install > Namespace > tomcat 입력 > Next 를 선택합니다.
- 170줄 변경 : service.type > ClusterIP
```bash
- service: 
    type: ClusterIP
```
- Install 을 선택합니다.

- Cluster > Services > tomcat > http 클릭합니다.
> Rancher에서 제공하는 Proxy를 이용하여 클러스터 내 서비스에 접속할 수 있습니다.  
> 접속하는 URL규칙은 api/v1/namespace/네임스페이스명/services/http:서비스명:서비스포트/proxy/ 
>> ex) https://rancher.kubeworks.net/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy/

---

**2) nginx 웹 서버 설치**

- Cluster > local > Worksloads > Create
- Deployment > Namespace > Create a New Namespace : nginx 입력합니다.
- Name : nginx > Container Image : nginx 를 입력합니다.
- Ports > ClusterIP > Name : http > Private Container Port : 80 > Create 를 선택합니다.


- Cluster > Services > nginx > http 클릭합니다. 
