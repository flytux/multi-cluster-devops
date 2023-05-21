### Lab 4. Argo workflow CI/CD


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

**2) Argo workflow 설치**

- Argo Workflow를 설치하고 접속 설정을 적용합니다.

~~~bash
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
~~~

**3) Run Sample WorkflowTemplate**

- SpringBoot 샘플 어플리케이션을 maven 빌드 후 컨테이너 이미지 저장소에 푸쉬 합니다.
- ArgoCD Gitops 레파지토지와 동기화하여 자동 배포를 적용합니다. (TBD)
- Git 인증 정보 / Docker Registry 인증 정보 추가 (TBD)
- Git Trigger를 통한 파이프라인 자동 구동 추가 (TBD)
- Approval / 메시지 통지등 관리 

```bash
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
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
          - name: image-tag
            value: newTag
      outputs: {}
      metadata: {}
      steps:
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
                  value: '{{inputs.parameters.image-tag}}'
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