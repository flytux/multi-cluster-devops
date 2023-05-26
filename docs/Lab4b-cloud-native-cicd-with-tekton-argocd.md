### Lab 4b. Cloud Native CI/CD with tekton and argocd

---

**1) Tekton Pipeline, Trigger, Dashboard 설치**

~~~bash
$ kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.48.0/release.yaml
$ kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/previous/v0.24.0/release.yaml 
$ kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/previous/v0.24.0/interceptors.yaml
$ kubectl apply -f https://github.com/tektoncd/dashboard/releases/download/v0.35.0/release-full.yaml

$ cat << EOF >> tekton-ing.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tekton-dash
  namespace: tekton-pipelines
spec:
  ingressClassName: nginx
  rules:
    - host: tekton.kw01
      http:
        paths:
          - backend:
              service:
                name: tekton-dashboard
                port:
                  number: 9097
            path: /
            pathType: Prefix
EOF
$ k apply -f tekton-ing.yml
~~~

- http://tekton.kw01

---

**2) 빌드 파이프라인 Task, Pipleines 설치**

~~~
$ k create ns build
$ k apply -f tekton-pipelines -n build
# ArgoCD 사용자 패스워드 입력합니다.
$ kubectl create secret generic -n build argocd-credentials-secret --from-literal=argocd-user-password='QHVwmkghjz1q7D2A' --from-literal=argocd-user-id='admin'

# tekton cli 설치합니다.
$ wget https://github.com/tektoncd/cli/releases/download/v0.31.0/tkn_0.31.0_Linux_x86_64.tar.gz
$ sudo tar xvf tkn_0.31.0_Linux_x86_64.tar.gz -C /usr/local/bin

$ k create -f tekton-pipelines/pr-kw-build.yml

# 파이프라인 구동 로그를 확인합니다.
$ tkn pr logs -f 
~~~

---
> To Many Open Files 오류 시   
>> sysctl -w fs.inotify.max_user_watches=100000   
>> sysctl -w fs.inotify.max_user_instances=100000   
---

- http://tekton.kw01/#/namespaces/build/pipelineruns

- http://10.214.156.101:30099/ 

---

**3) Gitea 레파지토리에 웹훅 추가**
- http://gitea.kw01/argo/kw-mvn
- Settings > Webhooks > Add Webhook > Gitea
- Target URL : http://el-build-listener.build:8080
- Add Webhook 을 선택합니다.
- Webhook > Test Delivery로 테스트 웹훅을 전송합니다.
- 파이프라인이 정상 기동하는지 확인합니다.

---

**4) Gitea 레파지토리 Push로 파이프라인 구동**
- 소스 파일을 변경 후 Push 합니다.
- 파이프라인 구동을 확인합니다.
- ArgoCD 앱과 랜처의 배포 상태를 확인합니다.
- 앱 접속 URL을 확인합니다.
- http://10.214.156.101:30099/
