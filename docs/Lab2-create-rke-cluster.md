**1) RKE2 cluster 설치**

- rke2 설치 스크립트를 이용하여 설치합니다.
- 인터넷 연결이 가능한 환경에서 실행합니다.

```bash
$ curl -sfL https://get.rke2.io | sh -

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
$ sed -i 's/default/rke2/g'  ~/.kube/config
```

