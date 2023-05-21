### Lab 4. Argo workflow CI/CD  

- Argo Workflow와 ArgoCD를 이용하여 Cloud Native CI/CD 환경을 구성합니다.
- Gitea - Argo Workflow - Docker Registry - ArgoCD - 배포 대상 클러스터 순서로 빌드 프로세스는 진행됩니다.
- Gitea 의 Event를 통해서 파이프라인 구동을 자동화 합니다.

---

**1) Docker Registry 설치**

- CI/CD 빌드 후 컨테이너 이미지를 저장할 docker registry를 설치합니다.
- 공인인증서 대신 insecure 설정을 적용하고 containerd에 인증 정보를 등록합니다.

```bash
# Docker Registry Helm Chart 설치
$ helm repo add twuni https://helm.twun.io
$ helm fetch twuni/docker-registry

$ cat << EOF >> value.yaml
service:
  name: registry
  type: NodePort
  port: 5000
  nodePort: 30005
persistence:
  accessMode: 'ReadWriteOnce'
  enabled: true
  size: 10Gi
  storageClass: 'local-path'
EOF

$ helm install docker-registry -f values.yaml docker-registry-2.2.2.tgz -n registry --create-namespace

$ curl -v localhost:30005/v2/_catalog

# admin / 1 로 로그인
$ nerdctl --insecure-registry login 10.214.156.72:30005

# nerdctl download
$ wget https://github.com/containerd/nerdctl/releases/download/v1.3.1/nerdctl-full-1.3.1-linux-amd64.tar.gz
$ tar Cxzvvf /usr/local nerdctl-full-1.3.1-linux-amd64.tar.gz

# nerdctl 옵션 설정
$ cat << EOF >> /etc/containerd/config.toml .
debug          = false
debug_full     = false
address        = "unix:///run/k3s/containerd/containerd.sock"
namespace      = "k8s.io"
cgroup_manager = "cgroupfs"
hosts_dir      = ["/etc/containerd/certs.d", "/etc/docker/certs.d"]
EOF

# 컨테이너 런타임에 Private Registry 인증 / insecure 설정
$ cat << EOF | sudo tee /etc/rancher/rke2/registries.yaml
"10.214.156.72:30005":
    endpoint:
      - "http://10.214.156.72:30005"
configs:
  "10.214.156.72:30005":
    auth:
      username: admin # this is the registry username
      password: 1 # this is the registry password
    tls:
      insecure_skip_verify: true
EOF
$ systemctl restart rke2-server

# 아래 파일에 insecure 및 인증 설정 추가 확인 
$ cat /var/lib/rancher/rke2/agent/etc/containerd/config.toml
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
- http://gitea.kw01 에 접속합니다.
- server domain과 접속 URL을 http://gitea.kw01 로 설정하고 저장합니다.
- http://gitea.kw01에 접속하여 신규 계정을 생성합니다.
- 사용자 ID : argo / 패스워드 : 12345678
- argo 계정으로 로그인하여 New Migration으로 레파지토리를 import 합니다.
- https://github.com/flytux/kw-mvn-deploy.git

---

**3) ArgoCD 설치**

```bash
# ArgoCD 설치
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Ingress Controller 설정 변경
$ kubectl edit ds -n kube-system rke2-ingress-nginx-controller

# 61 줄에 아래 내용 추가
    - --watch-ingress-without-class=true
    - --enable-ssl-passthrough
# 저장

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

# ArgoCD 초기 패스워드 확인
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
- https://argocd.kw01 admin / 초기 패스워드로 로그인합니다.

```bash
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
---

**3) Argo workflow 설치**

- Argo Workflow를 설치하고 접속 설정을 적용합니다.

```bash
# Install argo-workflow
$ kubectl create namespace argo
$ kubectl apply -f https://github.com/argoproj/argo-workflows/releases/download/v3.4.7/install.yaml

# Patch auth-mode
$ kubectl patch deployment \
  argo-server \
  --namespace argo \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
  "server",
  "--auth-mode=server"
]}]'

$ cat << EOF >> svc-patch.yml
spec:
  ports:
  - name: web
    port: 2746
    protocol: TCP
    targetPort: 2746
    nodePort: 30274
  type: NodePort
EOF

$ k patch -n argo svc argo-server --patch-file svc-patch.yml

# Download the binary
$ curl -sLO https://github.com/argoproj/argo-workflows/releases/download/v3.4.7/argo-linux-amd64.gz

# Unzip
$ gunzip argo-linux-amd64.gz

# Make binary executable
$ chmod +x argo-linux-amd64

# Move binary to path
$ mv ./argo-linux-amd64 /usr/local/bin/argo

# Test installation
$ argo version
```
---

**3) Run Sample WorkflowTemplate**

- SpringBoot 샘플 어플리케이션을 maven 빌드 후 컨테이너 이미지 저장소에 푸쉬 합니다.
- ArgoCD Gitops 레파지토지와 동기화하여 자동 배포를 적용합니다.
- Git 인증 정보 / Docker Registry 인증 정보 추가
- Git Trigger를 통한 파이프라인 자동 구동 추가 (TBD)
- Approval / 메시지 통지등 관리 

```bash
$ kubectl create secret generic gitops-secret --from-literal=gitops-repo-secret='http://argo:12345678@gitea.gitea:3000'
$ kubectl create secret generic argocd-credentials-secret --from-literal=argocd-user-password='e9P6sidSgm41J2lD' --from-literal=argocd-user-id='admin'

# WorkflowTemplate

metadata:
  name: mvn-build-kldgb
  generateName: mvn-build-
  namespace: argo
spec:
  templates:
    - name: mvn-build
      inputs:
        parameters:
          - name: git-url
            value: https://github.com/flytux/kw-mvn.git
          - name: revision
            value: main
          - name: image-url
            value: 10.214.156.72:30005/kw-mvn
          - name: stage
            value: dev
          - name: gitops-url
            value: http://gitea.gitea:3000/argo/kw-mvn-deploy.git
          - name: gitops-branch
            value: kust
          - name: argocd-app-name
            value: kw-mvn
          - name: argocd-url
            value: argocd-server.argocd
      outputs: {}
      metadata: {}
      steps:
        - - name: get-build-id
            template: build-id
            arguments:
              parameters:
                - name: stage
                  value: '{{inputs.parameters.stage}}'
        - - name: clone-sources
            template: git-clone
            arguments:
              parameters:
                - name: git-url
                  value: '{{inputs.parameters.git-url}}'
                - name: revision
                  value: '{{inputs.parameters.revision}}'
        - - name: build-push
            template: build-mvn
            arguments:
              parameters:
                - name: image-url
                  value: '{{inputs.parameters.image-url}}'
                - name: image-tag
                  value: '{{steps.get-build-id.outputs.parameters.build-id}}'
        - - name: argo-update
            template: update-manifest
            arguments:
              parameters:
                - name: gitops-url
                  value: '{{inputs.parameters.gitops-url}}'
                - name: gitops-branch
                  value: '{{inputs.parameters.gitops-branch}}'
                - name: image-tag
                  value: '{{steps.get-build-id.outputs.parameters.build-id}}'
        - - name: argo-sync
            template: sync-argo-app
            arguments:
              parameters:
                - name: argocd-app-name
                  value: '{{inputs.parameters.argocd-app-name}}'
                - name: argocd-url
                  value: '{{inputs.parameters.argocd-url}}'
    - name: build-id
      inputs:
        parameters:
          - name: stage
      outputs:
        parameters:
          - name: build-id
            valueFrom:
              path: /workspace/result
      metadata: {}
      script:
        name: ''
        image: bash:5.0.18
        command:
          - sh
        workingDir: /workspace
        resources: {}
        volumeMounts:
          - name: workdir
            mountPath: /workspace
        source: |
          ts=`date "+%y%m%d%H%M%S"`
          echo "Current Timestamp: ${ts}"
          id=`echo $RANDOM | md5sum | head -c 6`
          buildId={{inputs.parameters.stage}}-${ts}-${id}
          echo ${buildId} | tr -d "\n" | tee /workspace/result
    - name: git-clone
      inputs:
        parameters:
          - name: git-url
          - name: revision
      outputs: {}
      metadata: {}
      script:
        name: ''
        image: alpine/git:v2.26.2
        command:
          - sh
        workingDir: /workspace
        resources: {}
        volumeMounts:
          - name: workdir
            mountPath: /workspace
        source: |
          rm -rf *
          rm -rf .git
          rm -rf .mvn
          git init
          git remote add origin "{{inputs.parameters.git-url}}"
          git fetch --depth 1 origin "{{inputs.parameters.revision}}"
          git checkout "{{inputs.parameters.revision}}"
    - name: build-mvn
      inputs:
        parameters:
          - name: image-url
          - name: image-tag
      outputs: {}
      metadata: {}
      container:
        name: ''
        image: maven:3-jdk-11
        command:
          - mvn
        args:
          - '-B'
          - '-Duser.home=/workspace'
          - '-DsendCredentialsOverHttp=true'
          - '-Djib.allowInsecureRegistries=true'
          - >-
            -Djib.to.image={{inputs.parameters.image-url}}:{{inputs.parameters.image-tag}}
          - compile
          - com.google.cloud.tools:jib-maven-plugin:build
        workingDir: /workspace
        resources: {}
        volumeMounts:
          - name: workdir
            mountPath: /workspace
          - name: m2-cache
            mountPath: /workspace/.m2
    - name: update-manifest
      inputs:
        parameters:
          - name: gitops-url
          - name: gitops-branch
          - name: image-tag
      outputs: {}
      metadata: {}
      script:
        name: ''
        image: alpine/git:v2.26.2
        command:
          - sh
        workingDir: /workspace
        env:
          - name: GITOPS_REPO_CREDENTIALS
            valueFrom:
              secretKeyRef:
                name: gitops-secret
                key: gitops-repo-secret
        resources: {}
        source: |

          mkdir deploy && cd deploy
          git init  
          echo $GITOPS_REPO_CREDENTIALS >  ~/.git-credentials
          cat ~/.git-credentials  
          git config credential.helper store
          git remote add origin {{inputs.parameters.gitops-url}}  
          git remote -v 
          git -c http.sslVerify=false fetch --depth 1 origin {{inputs.parameters.gitops-branch}}  
          git checkout {{inputs.parameters.gitops-branch}}  
          ls -al 
          echo "updating image to {{inputs.parameters.image-tag}}" 
          sed -i "s|newTag:.*$|newTag: {{inputs.parameters.image-tag}}|" dev/kustomization.yaml 
          cat dev/kustomization.yaml | grep newTag 
          git config --global user.email "argo@devops" 
          git config --global user.name "Argo Workflow Pipelines"
          git add . 
          git commit --allow-empty -m "[argo] updating image to {{inputs.parameters.image-tag}}" 
          git -c http.sslVerify=false push origin {{inputs.parameters.gitops-branch}}
    - name: sync-argo-app
      inputs:
        parameters:
          - name: argocd-app-name
          - name: argocd-url
      outputs: {}
      metadata: {}
      script:
        name: ''
        image: quay.io/argoproj/argocd:v2.7.2
        command:
          - sh
        env:
          - name: ARGO_USER_ID
            valueFrom:
              secretKeyRef:
                name: argocd-credentials-secret
                key: argocd-user-id
          - name: ARGO_USER_PASSWORD
            valueFrom:
              secretKeyRef:
                name: argocd-credentials-secret
                key: argocd-user-password
        resources: {}
        source: >
          argocd login {{inputs.parameters.argocd-url}} --username $ARGO_USER_ID
          --password $ARGO_USER_PASSWORD --insecure

          argocd app sync {{inputs.parameters.argocd-app-name}} --insecure

          argocd app wait {{inputs.parameters.argocd-app-name}} --sync --health
          --operation --insecure
  entrypoint: mvn-build
  arguments: {}
  volumes:
    - name: workdir
      persistentVolumeClaim:
        claimName: pvc-argo-build-workspace
    - name: m2-cache
      persistentVolumeClaim:
        claimName: pvc-argo-build-cache

 ```
