### Lab 1. Create KubeADM Cluster

---

![https://kubernetes.io/docs/concepts/overview/components/](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)
- https://kubernetes.io/docs/concepts/overview/components/
---
![cluster-config](./cluster-config.png)

- kubeadm 클러스터 1 Master + 1 Worker 노드를 Terraform을 이용하여 설치합니다. (https://github.com/flytux/terraform-kube/tree/main/kubeadm)

---

**1) kubeadm cluster 설치**


- sudo 권한이 있는 사용자 계정과 4 Core, 16 Gi, 100 GB VM 2기를 준비합니다.
- ssh-key를 생성하여 각 VM에 root 계정에 authorized_key를 추가하여 ssh 로그인 이 가능하도록 설정합니다.
- terraform 모듈의 variables에 ssh-key 정보와 클러스터를 생성할 VM 정보를 편집하고 terraform apply를 실행하여 클러스터를 생성합니다.

```bash

# 클러스터를 설치할 bastion 환경에서 실행합니다. Bastion 서버가 없는 경우 생성한 VM 중 한개의 VM에서 실행합니다.

# terraform 모듈 clone
$ git clone https://github.com/flytux/terraform-kube && cd terraform-kube

# terraform download
$ curl -L https://github.com/opentofu/opentofu/releases/download/v1.7.0/tofu_1.7.0_linux_amd64.tar.gz | tar xvz -C /usr/local/bin
$ mv /usr/local/bin/tofu /usr/local/bin/tf

# 등록할 ssh-key 생성
$ ssh-keygen -b 2048 -t rsa -f .ssh/id_rsa.key -q -N ""

# 각 VM에 생성한 ssh-key 등록
# 각각의 VM에 생성한 .ssh/id_rsa.key.pub 파일을 root 계정의 authorized_key에 등록합니다.

# variable 파일 편집
$ cd kubeadm && vi variables.tf

# ssh_key 파일 경로 편집
variable "ssh_key" {default = ".ssh/id_rsa.key"}
variable "master_ip" { default = "192.168.122.11" } # 마스터 노드 VM의 IP 설정

variable "kubeadm_nodes" {

  type = map(object({ role = string, ip = string }))
  default = {
              kb-master-1 = { role = "master-init", ip = "192.168.122.11" },
              kb-worker-1 = { role = "worker", ip = "192.168.122.21" }, # 워커 노드 VM의 IP 설정
  }
}

# terraform apply 실행
tf destroy -auto-approve; tf apply -auto-approve

# 생성된 마스터 노드의 kubeconfig 파일을 계정의 kubeconfig에 추가합니다.
# kubeconfig의 클러스터 명을 mgmt로 변경합니다.
ssh -i ../kvm/.ssh-default/id_rsa.key -o StrictHostKeyChecking=no root@192.168.122.11 -- cat ~/.kube/config | sed  "s/kubernetes.*$/mgmt/g" > ~/.kube/config
```

- kubectl / helm cli을 설치합니다.
- shell 환경 변수에 alias를 설정합니다.

```bash
$ curl -LO https://dl.k8s.io/release/v1.28.8/bin/linux/amd64/kubectl
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
EOF

$ source ~/.bashrc

$ k get nodes -o wide
```

---

**2) 클러스터 버전 Upgrade**

- terraform kubeadm upgrade 모듈을 이용하여 업그레이드를 수행합니다.

```bash
# 현재 클러스터 버전 확인
$ k get nodes

# 마스터노드 버전 upgrade
$ cd ../kubeadm-upgrade

# 업그레이드 대상 노드 정보 편집
$ vi 01-variables.tf

variable "master_ip" { default = "192.168.122.11" }

variable "new_version" { default = "v1.28.9" }

variable "ssh_key" { default = "../kvm/.ssh-default/id_rsa.key" }

variable "kubeadm_nodes" {

  type = map(object({ role = string, ip = string}))
  default = {
              kubeadm-master-1 = { role = "master-init", ip = "192.168.122.11"},
              kubeadm-worker-1 = { role = "worker",      ip = "192.168.122.21"},
  }
}

# tf apply

$ tf destroy -auto-approve; tf apply -auto-approve

# 업그레이드 결과 확인
$ kubectl get nodes -A

```
