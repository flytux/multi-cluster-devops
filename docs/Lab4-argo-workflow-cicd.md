### Lab 4. Argo workflow CI/CD



**1) Argo workflow Install**


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

**2) Run Sample WorkflowTemplate**

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
