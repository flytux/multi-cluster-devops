### Lab 4. Argo workflow CI/CD



**1) Argo workflow**


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


