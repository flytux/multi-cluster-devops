### Lab 4b. Cloud Native CI/CD with tekton and argocd

**1) Tekton Pipeline, Trigger, Dashboard 설치

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

**2) Install Pipeline**

~~~
$ k create ns build
$ k apply -f tekton-pipelines -n build
$ wget https://github.com/tektoncd/cli/releases/download/v0.31.0/tkn_0.31.0_Linux_x86_64.tar.gz
$ sudo tar xvf tkn_0.31.0_Linux_x86_64.tar.gz -C /usr/local/bin
~~~

- Add Project "APPS" in Rancher
- Move build namespace to "APPS" project

~~~
$ k apply -f charts/tekton/argo-app-kw-mvn.yml
~~~

**6) Run Pipeline**
~~~
$ kcg
$ kc rke
$ kn build
$ k create -f charts/tekton/pipeline/pr-kw-build.yml
~~~

- http://tekton.vm01/#/namespaces/build/pipelineruns

&nbsp;

- Check application : http://vm01:30088/ 
  
**7) Add Webhook to Gitea Repo**
- http://gitea.vm01/tekton/kw-mvn
- Settings > Webhooks > Add Webhook > Gitea
- Target URL : http://el-build-listener.build:8080
- Add Webhook
- Click Webhook > Test Delivery
- Check Pipeline Runs

&nbsp;

**8) Git push source repo will trigger tekton pipeline**
- Edit source and commit
- Check Pipeline Runs
- Check argocd app deployment status
- Check application : http://vm01:30088/ 

&nbsp;

