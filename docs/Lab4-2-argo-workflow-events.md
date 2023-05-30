### Lab 4-2. Argo workflow and events

- Argo workflow를 이용하여 빌드 파이프라인을 구성합니다.
- Argo event를 통해 Git 웹훅을 받아 파이프라인을 기동합니다.


<img title="gitea-argo-workflow" alt="gitea-argo" src="./gitea-argo-workflow.png" >

---

**1) Argo workflow 설치**

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
```
---

**2) 빌드 파이프라인 준비**

- SpringBoot 샘플 어플리케이션을 maven 빌드 후 컨테이너 이미지 저장소에 푸쉬 합니다.
- ArgoCD Gitops 레파지토지와 동기화하여 자동 배포를 적용합니다.
- Gitea와 ArgoCD 인증정보는 argo 네임스페이스에 secret으로 설정하여 적용합니다.

```bash
# 이전에 등록한 gitea 계정 / 패스워드 정보와 Argocd 패스워드 정보를 Secret으로 생성합니다.
$ kubectl create secret generic -n argo gitops-secret --from-literal=gitops-repo-secret='http://argo:12345678@gitea.gitea:3000'
# ArgoCD 패스워드는 앞에서 확인한 패스워드를 입력합니다.
$ kubectl create secret generic -n argo argocd-credentials-secret --from-literal=argocd-user-password='7NKSA3w19yQ4XGAL' --from-literal=argocd-user-id='admin'

# 파이프라인용 pvc 생성
$ cat << EOF >> pvc-argo.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-argo-build-workspace
  namespace: argo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-path
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-argo-build-cache
  namespace: argo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-path
EOF

$ kubectl apply -f pvc-argo.yml -n argo
```
---

**3) 파이프라인 워크플로우 (WorkflowTemplate) 등록**

- Argo workflow 는 NodePort 서비스를 통해 접속합니다.
- http://노드IP:30274

```bash
# WorkflowTemplate Revised
# Argo Workflow UI에 접속하여 Workflow Template 생성 메뉴에서 아래 template를 하나씩 생성합니다.
---
metadata:
  name: build-id
  namespace: argo
spec:
  templates:
    - name: build-id
      inputs:
        parameters:
          - name: stage
      outputs:
        parameters:
          - name: build-id
            valueFrom:
              path: /workspace/result
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
  serviceAccountName: argo-pipeline-runner
---
metadata:
  name: git-clone
  namespace: argo
spec:
  templates:
    - name: git-clone
      inputs:
        parameters:
          - name: git-url
          - name: revision
      script:
        image: alpine/git:v2.26.2
        command:
          - sh
        workingDir: /workspace
        volumeMounts:
          - name: workdir
            mountPath: /workspace
        source: |
          rm -rf *
          rm -rf .git
          rm -rf .mvn
          git init
          git remote add origin {{inputs.parameters.git-url}}  
          git fetch --depth 1 origin {{inputs.parameters.revision}} 
          BRANCH=$(echo {{inputs.parameters.revision}} | cut -d/ -f3)
          git checkout $BRANCH
  serviceAccountName: argo-pipeline-runner
---
metadata:
  name: build-mvn
  namespace: argo
spec:
  templates:
    - name: build-mvn
      inputs:
        parameters:
          - name: image-url
          - name: image-tag
      container:
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
        volumeMounts:
          - name: workdir
            mountPath: /workspace
          - name: m2-cache
            mountPath: /workspace/.m2
  serviceAccountName: argo-pipeline-runner
---
metadata:
  name: update-manifest
  namespace: argo
spec:
  templates:
    - name: update-manifest
      inputs:
        parameters:
          - name: gitops-url
          - name: gitops-branch
          - name: image-tag
      script:
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
          git -c http.sslVerify=false fetch --depth 1 origin  
          {{inputs.parameters.gitops-branch}}  
          git checkout {{inputs.parameters.gitops-branch}}  
          echo "updating image to {{inputs.parameters.image-tag}}"  
          sed -i "s|newTag:.*$|newTag: {{inputs.parameters.image-tag}}|" dev/kustomization.yaml    
          cat dev/kustomization.yaml | grep newTag  
          git config --global user.email "argo@devops"  
          git config --global user.name "Argo Workflow Pipelines"  
          git add .  
          git commit --allow-empty -m "[argo] updating image to {{inputs.parameters.image-tag}}" 
          git -c http.sslVerify=false push origin {{inputs.parameters.gitops-branch}}  
  serviceAccountName: argo-pipeline-runner
---
metadata:
  name: sync-argo-app
  namespace: argo
spec:
  templates:
    - name: sync-argo-app
      inputs:
        parameters:
          - name: argocd-app-name
          - name: argocd-url
      script:
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
        source: >
          argocd login {{inputs.parameters.argocd-url}} --username $ARGO_USER_ID
          --password $ARGO_USER_PASSWORD --insecure

          argocd app sync {{inputs.parameters.argocd-app-name}} --insecure

          argocd app wait {{inputs.parameters.argocd-app-name}} --sync --health
          --operation --insecure
  serviceAccountName: argo-pipeline-runner
---
metadata:
  name: mvn-build-webhook
  generateName: mvn-build-webhook-
  namespace: argo
spec:
  templates:
    - name: mvn-build
      inputs:
        parameters:
          - name: git-url            
            value: http://gitea.gitea:3000/argo/
          - name: revision        
            value: main
          - name: image-url
            value: 마스터노드IP:30005/kw-mvn
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
            arguments:
              parameters:
                - name: stage
                  value: '{{inputs.parameters.stage}}'
            templateRef:
              name: build-id
              template: build-id
        - - name: clone-sources
            arguments:
              parameters:
                - name: git-url
                  value: '{{inputs.parameters.git-url}}'
                - name: revision
                  value: '{{inputs.parameters.revision}}'
            templateRef:
              name: git-clone
              template: git-clone
        - - name: build-push
            arguments:
              parameters:
                - name: image-url
                  value: '{{inputs.parameters.image-url}}'
                - name: image-tag
                  value: '{{steps.get-build-id.outputs.parameters.build-id}}'
            templateRef:
              name: build-mvn
              template: build-mvn
        - - name: argo-update
            arguments:
              parameters:
                - name: gitops-url
                  value: '{{inputs.parameters.gitops-url}}'
                - name: gitops-branch
                  value: '{{inputs.parameters.gitops-branch}}'
                - name: image-tag
                  value: '{{steps.get-build-id.outputs.parameters.build-id}}'
            templateRef:
              name: update-manifest
              template: update-manifest
        - - name: argo-sync
            arguments:
              parameters:
                - name: argocd-app-name
                  value: '{{inputs.parameters.argocd-app-name}}'
                - name: argocd-url
                  value: '{{inputs.parameters.argocd-url}}'
            templateRef:
              name: sync-argo-app
              template: sync-argo-app
  entrypoint: mvn-build
  arguments:
    parameters:
      - name: repository-name
        value: kw-mvn
  serviceAccountName: argo-pipeline-runner
  volumes:
    - name: workdir
      persistentVolumeClaim:
        claimName: pvc-argo-build-workspace
    - name: m2-cache
      persistentVolumeClaim:
        claimName: pvc-argo-build-cache
---
```
- Argo Pipeline 실행을 위한 ServiceAccount와 권한을 설정합니다.

```bash
$ cat << EOF >> argo-rbac.yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argo-pipeline-runner
  namespace: argo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argo-pipeline-runner
rules:
  - apiGroups:
      - argoproj.io
    verbs:
      - "*"
    resources:
      - workflows
      - clusterworkflowtemplates
      - workflowtemplates
  - apiGroups:
      - ''
    resources:
      - 'pods'
    verbs:
      - 'create'
      - 'delete'
      - 'get'
      - 'list'
      - 'patch'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argo-pipeline-runner
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argo-pipeline-runner
subjects:
  - kind: ServiceAccount
    name: argo-pipeline-runner
    namespace: argo
EOF

$ kubectl apply -f argo-rbac.yml -n argo
```

- Workflow Template을 Submit 하여 빌드 프로세스를 구동합니다.
- https://마스터노드IP:30274/workflow-templates/argo/mvn-build-webhook-simple
- Submit > Entropint : mvn-build > image-url : 자신의 레지스트리 주소로 변경 >Submit

- Workflow 구동 결과와 ArgoCD의 동기화 결과, Rancher의 파드 구동 현황을 확인하고
- endpoint URL에 접속합니다 (http://마스터노드IP:30099/)

---

**4) Argo Events 설치**

```bash
# 네임스페이스, Argo Event 
$ helm repo add argo https://argoproj.github.io/argo-helm
$ helm install argo-events argo/argo-events -n argo
$ kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/eventbus/native.yaml -n argo
```
---

**5) Trigger 설정**

```bash
# 소스 레파지토리에서 main 브랜치 푸쉬시 웹훅을 통해 파이프라인이 기동되도록 설정합니다.
$ cat << EOF >> gitea-trigger.yml
---
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: gitea-event-source
spec:
  type: "webhook"
  service:
    ports:
      - port: 12000
        targetPort: 12000
  webhook:
    gitea-push:
      port: "12000"
      endpoint: "/push"
      method: "POST"
      filter:
        expression: "(body.ref == 'refs/heads/main')"
---
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: gitea-push
spec:
  dependencies:
    - name: build-dep
      eventSourceName: gitea-event-source
      eventName: gitea-push
  triggers:
    - template:
        name: gitea-push
        argoWorkflow:
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: build-trigger-
              spec:
                arguments:
                  parameters:
                    - name: git-url
                      value: http://gitea.gitea:3000/argo/kw-mvn.git
                    - name: revision
                      value: main
                    - name: image-url
                      value: 10.214.156.101:30005/kw-mvn
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

                workflowTemplateRef:
                  name: mvn-build-webhook
          operation: submit
          parameters:
            - src:
                dependencyName: build-dep
                dataKey: body.repository.name
              dest: spec.arguments.parameters.0.value
              operation: append
  template:
    serviceAccountName: argo-pipeline-runner
EOF

$ kubectl apply -f gitea-trigger.yml -n argo
```
---

- Gitea의 소스 레파지토리에 웹훅을 등록합니다.
- https://gitea.kw01/argo/kw-mvn/settings/hooks
- Add Webhook > Gitea 
- Target URL 설정 : http://gitea-event-source-eventsource-svc.argo:12000/push
- Test Delivery 클릭하여 웹훅 동작을 확인합니다.
- gitea.kw01/argo/kw-mvn의 main 브랜치를 Push하여 파이프라인 기동을 확인합니다.

---
