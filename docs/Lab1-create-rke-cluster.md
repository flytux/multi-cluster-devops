**1) RKE2 cluster 설치**

- rke2 설치 스크립트를 이용하여 설치합니다.
- 인터넷 연결이 가능한 환경에서 실행합니다.
- sudo 권한이 있는 사용자 계정과 4 Core, 16 Gi, 100 GB VM 3기를 준비합니다.

```bash

# vm에 로그인 후
$ sudo -i
$ curl -sfL https://get.rke2.io | sh -

# (Optional)
# 클러스터 버전 지정 시 INSTALL_RKE2_VERSION 지정
# Rancher 2.7.3 버전은 RKE2 1.26 이하 지원
$ curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.25.9+rke2r1 sh -
```

- systemctl 로 rke2-server 서비스를 기동합니다.
- 설치 중 systemctl로 rke2-server 서비스 상태를 확인합니다.
- 설치 완료 후 kubeconfig 파일을 기본 위치에 복사합니다.


```bash
$ sudo -i
$ systemctl enable rke2-server --now
$ systemctl status -l rke2-server
$ journalctl -fa

$ su - k8sadm # 사용자 계정
$ mkdir ~/.kube
$ sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
$ sudo chown k8sadm ~/.kube/config
$ sed -i 's/default/rke2/g'  ~/.kube/config
```

- kubectl / helm cli을 설치합니다.
- shell 환경 변수에 alias를 설정합니다.

```bash
$ curl -LO https://dl.k8s.io/release/v1.25.9/bin/linux/amd64/kubectl
$ chmod +x kubectl && sudo mv kubectl /usr/local/bin
$ curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

$ cat <<EOF >> ~/.bashrc
# k8s alias
source <(kubectl completion bash)
complete -o default -F __start_kubectl k

alias k=kubectl
alias kn='kubectl config set-context --current --namespace'
alias kc='kubectl config use-context'
alias kcg='kubectl config get-contexts'
alias di='docker images --format "table {{.Repository}}:{{.Tag}}\t{{.ID}}\t{{.Size}}\t{{.CreatedSince}}"'
alias kge="kubectl get events  --sort-by='.metadata.creationTimestamp'  -o 'go-template={{range .items}}{{.involvedObject.name}}{{\"\t\"}}{{.involvedObject.kind}}{{\"\t\"}}{{.message}}{{\"\t\"}}{{.reason}}{{\"\t\"}}{{.type}}{{\"\t\"}}{{.firstTimestamp}}{{\"\n\"}}{{end}}'"
EOF

$ source ~/.bashrc
```
