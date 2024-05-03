### Lab 4-1 Cloud Native CI/CD Toolchains

- Argo Workflow와 ArgoCD를 이용하여 Cloud Native CI/CD 환경을 구성합니다.
- Git clone -> maven build -> image push -> argo-deploy 순서로 빌드 프로세스는 진행됩니다.
- Gitea 의 Event를 통해서 파이프라인 구동을 자동화 합니다.

---
![CI/CD 파이프라인](./cicd-pipelines.png)
---

**1) Docker Registry 설치**

- CI/CD 빌드 후 컨테이너 이미지를 저장할 docker registry를 설치합니다.
- 사설인증서를 생성하고 containerd에 인증 정보를 등록합니다.

```bash

# 도커 레지스트리 용 사설 인증서 생성
$ openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -keyout docker.key -out docker.crt -subj '/CN=docker.kw01' \
  -addext 'subjectAltName=DNS:docker.kw01'

# Docker Registry Helm Chart 설치
$ helm repo add twuni https://helm.twun.io

$ kubectl create ns registry
$ kubectl create secret tls docker-ingress-tls --key docker.key --cert docker.crt -n registry

$ cat << EOF >> values.yaml
ingress:
  enabled: true
  className: nginx
  path: /
  hosts:
    - docker.kw01
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
  tls:
    - secretName: docker-ingress-tls
      hosts:
        - docker.kw01
persistence:
  accessMode: 'ReadWriteOnce'
  enabled: false
  size: 10Gi
  storageClass: 'local-path'

EOF

$ helm install docker-registry -f values.yaml twuni/docker-registry -n registry 

# registry login 테스트
$ cp docker.key docker.crt /usr/local/share/ca-certificates/
$ update-ca-certificates

# nerdctl download
$ wget https://github.com/containerd/nerdctl/releases/download/v1.3.1/nerdctl-full-1.3.1-linux-amd64.tar.gz
$ sudo tar Cxzvvf /usr/local nerdctl-full-1.3.1-linux-amd64.tar.gz

# admin / 1 로 로그인
$ nerdctl loing docker.kw01 # hosts 파일에 dns entry 추가

# 컨테이너 런타임에 인증서 추가 (마스터/워코 모두 추가)
$ scp -i ../terraform-kube/kvm/.ssh-default/id_rsa.key -o StrictHostKeyChecking=no docker.key root@192.168.122.11:/root
$ scp -i ../terraform-kube/kvm/.ssh-default/id_rsa.key -o StrictHostKeyChecking=no docker.crt root@192.168.122.11:/root
$ ssh -i ../terraform-kube/kvm/.ssh-default/id_rsa.key -o StrictHostKeyChecking=no root@192.168.122.11

$ cp docker* /etc/pki/ca-trust/source/anchors/
$ update-ca-trust
# admin / 1 로 로그인
$ nerdctl loing docker.kw01 # hosts 파일에 dns entry 추가

$ vi /etc/containerd/containerd.toml

[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
   [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
      [plugins."io.containerd.grpc.v1.cri".registry]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
          [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.kw01"]
            endpoint = ["https://docker.kw01"]
            [plugins."io.containerd.grpc.v1.cri".registry.configs."docker.kw01".tls]
              ca_file = "/etc/pki/ca-trust/source/anchors/docker.crt"
              cert_file = "/etc/pki/ca-trust/source/anchors/docker.crt"
              key_file = "/etc/pki/ca-trust/source/anchors/docker.key"

$ systemctl restart containerd

```

---

**2) Gitea 설치**

```bash
$ kubectl apply -f - <<"EOF"
apiVersion: v1
kind: Namespace
metadata:
  name: "gitea"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: gitea
  name: gitea
  labels:
    app: gitea
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitea
  template:
    metadata:
      labels:
        app: gitea
    spec:
      containers:
      - name: gitea
        image: gitea/gitea:1.16
        env:
        - name: GITEA__webhook__ALLOWED_HOST_LIST
          value: '*'
        ports:
        - containerPort: 3000
          name: gitea
        - containerPort: 22
          name: git-ssh
        volumeMounts:
        - mountPath: /data
          name: git-volume
      volumes:
      - name: git-volume
        persistentVolumeClaim:
          claimName: git-volume
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: gitea
  name: git-volume
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  namespace: gitea
  name: gitea
spec:
  type: ClusterIP
  selector:
    app: gitea
  ports:
    - port: 3000
      targetPort: 3000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea
  namespace: gitea
spec:
  ingressClassName: nginx
  rules:
  - host: gitea.kw01
    http:
      paths:
      - backend:
          service:
            name: gitea
            port:
              number: 3000
        path: /
        pathType: Prefix
EOF
```

> host 파일에 gitea.kw01 argocd.kw01을 추가합니다.  
> rancher와 동일한 형태로 추가하면 됩니다.  
> 10.178.0.25 rancher.kw01 argocd.kw01 argo.kw01 gitea.kw01   #자신의 NODE1 IP

- http://gitea.kw01 에 접속합니다.
- server domain과 접속 URL을 http://gitea.kw01 로 설정하고 저장합니다.
- http://gitea.kw01에 접속하여 신규 계정을 생성합니다.
- 사용자 ID : argo / 패스워드 : 12345678
- argo 계정으로 로그인하여 New Migration으로 레파지토리를 import 합니다.
- https://github.com/flytux/kw-mvn.git
- https://github.com/flytux/kw-mvn-deploy.git

---

**3) ArgoCD 설치**

```bash
# ArgoCD 설치
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Ingress Controller 설정 변경
$ k edit deploy -n ingress-nginx ingress-nginx-controller

# 63 줄에 아래 내용 추가 : - --enable-ssl-passthrough
    - --enable-metrics=false
    - --enable-ssl-passthrough # 추가 내용
# 저장 :wq!

# deployment 재기동
$ k scale deploy ingress-nginx-controller --replicas=0
$ k scale deploy ingress-nginx-controller --replicas=1


# Ingress 설정
$ kubectl -n argocd apply -f - <<"EOF"  
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: argocd.kw01
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
EOF

# argocd cli 다운로드
$ wget https://github.com/argoproj/argo-cd/releases/download/v2.10.9/argocd-linux-amd64
$ mv argocd-linux-amd64 /usr/local/bin/argocd && chmod +x /usr/local/bin/argocd

# ArgoCD 초기 패스워드 확인
$ argocd admin initial-password -n argocd


```bash
# Default Project 등록

$ kubectl -n argocd apply -f - <<"EOF"
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
  namespace: argocd
spec:
  sourceRepos:
    - '*'
  destinations:
    - namespace: '*'
      server: '*'
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
EOF

# ArgoCD App 등록

$ kubectl -n argocd apply -f - <<"EOF"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kw-mvn
  namespace: argocd
spec:
  destination:
    namespace: deploy
    server: https://kubernetes.default.svc
  project: default
  source:
    path: dev
    repoURL: http://gitea.gitea:3000/argo/kw-mvn-deploy
    targetRevision: kust
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
EOF
```

- 설치된 Argo App의 컨테이너이미지 주소를 설치한 Docker Registry 정보에 맞추어 변경합니다.
- https://gitea.kw01/argo/kw-mvn-deploy/src/branch/kust/base/deploy.yml
- 19번째 줄 아래와 같이 변경 후 Commit 합니다.

```bash
  # 설치된 자신의 Docker Registry 주소로 변경합니다.
  image: docker.kw01/kw-mvn:init
  # 저장후 commit 합니다.
```
- https://gitea.kw01/argo/kw-mvn-deploy/src/branch/kust/dev/kustomization.yaml
- 9번째 줄 아래와 같이 변경 후 Commit 합니다.

```bash
  # 위와 동일하게 Docker Registry 주소로 변경합니다.
  - name: docker.kw01/kw-mvn
  # 저장후 commit 합니다.
```  
---

**4) ArgoCD를 이용한 배포 자동화**
- 설치된 ArgoCD는 GitOps 레파지토리를 기준으로 변경된 형상을 동기화 합니다.
- ArgoCD 앱에는 Gitops 레파지토리 정보와 배포될 클러스터 정보가 포함되어 있습니다.
- 자동 동기화 설정 시 2분에 한번씩 레파지토리 변경 내용을 반영합니다.

```bash
# Git Source를 클론합니다.

$ cat << EOF | sudo tee -a /etc/hosts
$MY_NODE1_IP gitea.kw01
EOF

$ git clone http://gitea.kw01/argo/kw-mvn 
$ cd kw-mvn

# maven 빌드를 이용하여 컨테이너 이미지를 생성합니다.
$ sudo nerdctl run -it --rm \
  -v "$(pwd)":/usr/src/kw-mvn -w /usr/src/kw-mvn \
  --add-host=docker.kw01:192.168.122.11 \
  maven:3.3-jdk-8 mvn clean compile jib:build \
  -Dimage=docker.kw01/kw-mvn:init \
  -DsendCredentialsOverHttp=true \
  -Djib.to.auth.username=admin \
  -Djib.to.auth.password=1 \
  -Djib.allowInsecureRegistries=true

  
# 생성된 이미지 태그를 확인합니다.
$ curl -L docker.kw01/v2/kw-mvn/tags/list

# Argo GitOps Repositoty의 이미지 Tag를 init으로 변경하고 Commit 합니다.
# http://gitea.kw01/argo/kw-mvn-deploy/src/branch/kust/dev/kustomization.yaml 10번째 줄

images:
- name: 10.178.0.25:30005/kw-mvn
  newTag: init

# ArgoCD에 로그인 해서 App을 동기화 합니다.
# https://argocd.kw01/applications/argocd/kw-mvn?view=tree&resource=
# Sync > Syncronize

# App에 접속합니다.
# http://192.168.122.11:30099


