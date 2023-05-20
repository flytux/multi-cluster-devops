### Lab 4. Argo workflow CI/CD


**1) Docker Registry 설치**

```bash
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

# 컨테이너 런타임에 Private Registry 인증 / insecure 설정
$ cat << EOF | sudo tee /etc/rancher/rke2/registries.yaml
mirrors:
  docker.io:
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
```

**2) Argo workflow 설치**


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

```bash
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: mvn-build-
spec:
  entrypoint: mvn-build
  volumes:
  - name: workdir
    persistentVolumeClaim:
      claimName: pvc-test

  templates:
  - name: mvn-build
    inputs:
      parameters:
        - name: git-url
          value: https://github.com/flytux/kw-mvn.git
        - name: revision
          value: main
        - name: image-url
          value: kw01
        - name: image-tag
          value: newTag
    steps:
    - - name: clone-sources
        template: git-clone
        arguments:
          parameters:
            - name: git-url
              value: "{{inputs.parameters.git-url}}"
            - name: revision
              value: "{{inputs.parameters.revision}}" 
    - - name: build
        template: build-mvn
        arguments:
          parameters:
            - name: image-url
              value: "{{inputs.parameters.image-url}}"
            - name: image-tag
              value: "{{inputs.parameters.image-tag}}" 
  
  - name: git-clone
    inputs:
      parameters:
      - name: git-url
      - name: revision
    script:      
      image: alpine/git:v2.26.2
      workingDir: /workspace
      command: [sh]
      source: |
        rm -rf *
        rm -rf ./.[!.]*
        rm -rf //..?*
        git init
        git remote add origin "{{inputs.parameters.git-url}}"
        git fetch --depth 1 origin "{{inputs.parameters.revision}}"
        git checkout "{{inputs.parameters.revision}}"
      volumeMounts:
      - name: workdir
        mountPath: /workspace    
        
  - name: build-mvn
    inputs:
      parameters:
      - name: image-url
      - name: image-tag
    container:      
      image: maven:3-jdk-11
      workingDir: /workspace
      command: ["mvn"]
      args: ["clean", "compile"]
      volumeMounts:
      - name: workdir
        mountPath: /workspace    
 ```
