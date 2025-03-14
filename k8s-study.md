# AWS EC2에 Kubernetes 클러스터 구축 가이드

## 목차
- [목표](#목표)
- [사전 준비](#사전-준비)
- [EC2 초기 세팅](#ec2-초기-세팅)
- [클러스터 구성](#클러스터-구성)
- [테스트 애플리케이션 배포](#테스트-애플리케이션-배포)

## 목표
EC2에 Kubernetes 클러스터를 구축합니다.
참고: https://bydawn25.tistory.com/78

## Next 목표
NLB, MetalLB 등으로 ingress controller 구성하기

## 사전 준비
### VPC 구성
### EC2 인스턴스 생성
- **개수**: 4개
- **유형**: t3.large
- **OS**: Ubuntu 22.04
- **IAM 역할**: SSM

## EC2 초기 세팅

### AWS CLI 설치
루트 권한으로 전환 후 AWS CLI를 설치합니다.
```bash
sudo -i
apt update && apt install -y awscli
```

### 호스트명 설정
IMDSv2를 사용해 인스턴스의 Name 태그를 가져와 호스트명으로 설정합니다.
```bash
sudo -i
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600") && INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id) && REGION=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/region) && NAME_TAG=$(aws ec2 describe-tags --region $REGION --filters "Name=resource-id,Values=$INSTANCE_ID" "Name=key,Values=Name" --query "Tags[0].Value" --output text)
hostnamectl set-hostname "$NAME_TAG"
```

### 스왑 비활성화
Kubernetes는 성능 예측을 위해 스왑을 비활성화해야 합니다.
```bash
sudo -i
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
> 이 명령어는 /etc/fstab 파일에서 "swap"이 포함된 줄을 찾아 주석 처리하여 재부팅 후에도 스왑이 비활성화된 상태로 유지합니다.

### 커널 모듈 설정
Kubernetes에 필요한 커널 모듈을 설정합니다.

#### 커널 모듈 설정 파일 생성
```bash
sudo tee /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF
```

#### 커널 모듈 로드
```bash
sudo modprobe overlay && sudo modprobe br_netfilter
```

#### 커널 모듈 로드 확인
```bash
lsmod | grep -E 'br_netfilter|overlay'
```

### 커널 파라미터 설정
Kubernetes 네트워킹에 필요한 커널 파라미터를 설정합니다.

#### 커널 파라미터 설정 파일 생성
```bash
sudo tee /etc/sysctl.d/k8s.conf << 'EOF'
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

#### 커널 파라미터 적용
```bash
sudo sysctl --system
```

### containerd 설치
컨테이너 런타임으로 containerd를 설치합니다.

#### containerd 다운로드 및 설치
```bash
sudo -i
wget https://github.com/containerd/containerd/releases/download/v1.6.9/containerd-1.6.9-linux-amd64.tar.gz && sudo tar Cxzvf /usr/local containerd-1.6.9-linux-amd64.tar.gz
```

#### containerd 서비스 등록
```bash
sudo mkdir -p /usr/local/lib/systemd/system && sudo wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -O /usr/local/lib/systemd/system/containerd.service && sudo systemctl daemon-reload && sudo systemctl enable --now containerd && systemctl status containerd
```

### runc 설치
OCI 런타임 표준 구현체인 runc를 설치합니다.
```bash
wget https://github.com/opencontainers/runc/releases/download/v1.1.3/runc.amd64 && sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

### CNI 플러그인 설치
컨테이너 네트워킹을 위한 CNI 플러그인을 설치합니다.
```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz && sudo mkdir -p /opt/cni/bin && sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

### containerd를 CRI 런타임으로 사용하기 위한 설정
containerd를 Kubernetes의 CRI(Container Runtime Interface)로 사용하기 위한 설정을 변경합니다.
```bash
sudo mkdir -p /etc/containerd && sudo containerd config default | sudo tee /etc/containerd/config.toml && sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml && sudo systemctl restart containerd
```

### Kubernetes 패키지 설치
kubeadm, kubelet, kubectl을 설치하고 버전을 고정합니다.
```bash
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg && echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list && sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl && sudo apt-mark hold kubelet kubeadm kubectl
```

## 클러스터 구성

### 마스터 노드 초기화
마스터 노드에서 쿠버네티스 클러스터를 초기화합니다.
```bash
sudo kubeadm init --control-plane-endpoint=k8smaster --apiserver-advertise-address=10.0.9.19 --pod-network-cidr=10.244.0.0/16
```

마스터 노드에서 kubectl 사용을 위한 환경 설정:
```bash
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> ~/.bashrc
source ~/.bashrc
```

### 워커 노드 조인
워커 노드에서 마스터 노드 호스트명 설정:
```bash
echo "10.0.9.19 k8smaster" | sudo tee -a /etc/hosts
```

워커 노드를 클러스터에 조인:
```bash
sudo kubeadm join k8smaster:6443 --token vd1ppv.2g3q2tm6nxfmsnir \
        --discovery-token-ca-cert-hash sha256:c5a0aa585806ba0a36d61b34b4881aeecdddf3e451570878045de01827333a25
```

### 네트워크 플러그인 설치
마스터 노드에서 Flannel 네트워크 플러그인을 설치합니다.
```bash
wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml
```

## 테스트 애플리케이션 배포

### NGINX 파드 배포
마스터 노드에서 NGINX 파드를 배포합니다.
```bash
kubectl run nginx --image=nginx
```

배포된 파드 확인:
```bash
kubectl get pods -o wide
```

### NGINX 서비스 노출
NGINX 파드를 NodePort 서비스로 노출시킵니다.
```bash
kubectl expose pod nginx --port=80 --type=NodePort --name=nginx-service
```

생성된 서비스 확인:
```bash
kubectl get svc
```

노출된 서비스는 모든 노드의 할당된 NodePort(예: 32717)를 통해 접근할 수 있습니다:
```
http://<워커노드_퍼블릭IP>:32717
```

> 주의: EC2 보안 그룹에서 해당 포트(32717)에 대한 인바운드 트래픽을 허용해야 합니다.




# Kubernetes 마스터 노드 설치 로그 분석

## 전체 설치 로그

```bash
root@k8s-master:~# sudo kubeadm init --control-plane-endpoint=k8smaster --apiserver-advertise-address=10.0.9.19 --pod-network-cidr=10.244.0.0/16

I0314 13:17:06.825986    6720 version.go:256] remote version is much newer: v1.32.3; falling back to: stable-1.28
[init] Using Kubernetes version: v1.28.15
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W0314 13:17:07.329350    6720 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master k8smaster kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.9.19]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [10.0.9.19 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [10.0.9.19 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 8.003978 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: vd1ppv.2g3q2tm6nxfmsnir
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join k8smaster:6443 --token vd1ppv.2g3q2tm6nxfmsnir \
        --discovery-token-ca-cert-hash sha256:c5a0aa585806ba0a36d61b34b4881aeecdddf3e451570878045de01827333a25 \
        --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8smaster:6443 --token vd1ppv.2g3q2tm6nxfmsnir \
        --discovery-token-ca-cert-hash sha256:c5a0aa585806ba0a36d61b34b4881aeecdddf3e451570878045de01827333a25
...
```

### 1. kubeadm

이 가이드는 Kubernetes 클러스터의 마스터 노드를 `kubeadm`을 사용하여 초기화하는 과정을 설명합니다. 주요 목적은 클러스터의 중앙 제어 지점을 설정하는 것입니다.

### 2. kubeadm 명령어 상세 설명

```bash
sudo kubeadm init \
  --control-plane-endpoint=k8smaster \
  --apiserver-advertise-address=10.0.9.19 \
  --pod-network-cidr=10.244.0.0/16
```

- `--control-plane-endpoint`: 클러스터의 엔드포인트 주소를 지정합니다. 여기서는 `k8smaster`로 설정되었습니다.
- `--apiserver-advertise-address`: API 서버의 광고 주소로, 10.0.9.19로 구성되었습니다.
- `--pod-network-cidr`: Pod의 네트워크 대역을 10.244.0.0/16으로 설정합니다.


### 3. 상세 로그 분석

#### 로그 분석 전체 과정

```bash
I0314 13:17:06.825986    6720 version.go:256] remote version is much newer: v1.32.3; falling back to: stable-1.28
```
- **날짜/타임스탬프**: `I0314 13:17:06.825986`
- **프로세스 ID**: `6720`
- **위치**: `version.go:256`
- **메시지**: 최신 버전(v1.32.3)이 있지만, 안정 버전(stable-1.28)으로 폴백

```bash
[init] Using Kubernetes version: v1.28.15
```
- 최종적으로 Kubernetes v1.28.15 버전 사용 결정

```bash
[preflight] Running pre-flight checks
```
- **사전 점검(Preflight Checks) 시작**
- 클러스터 초기화 전 시스템 환경 및 요구사항 확인

```bash
[preflight] Pulling images required for setting up a Kubernetes cluster
```
- 클러스터 설정에 필요한 Docker 이미지 다운로드 시작

```bash
[preflight] This might take a minute or two, depending on the speed of your internet connection
```
- 이미지 다운로드에 1-2분 소요될 수 있음을 사용자에게 안내

```bash
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
```
- 미리 `kubeadm config images pull` 명령어로 이미지를 다운로드할 수 있음을 안내

```bash
W0314 13:17:07.329350    6720 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
```
- **경고(Warning)**: 현재 샌드박스 이미지(pause:3.6)와 kubeadm 권장 이미지(pause:3.9) 불일치
- CRI(Container Runtime Interface) 샌드박스 이미지 업데이트 권장

```bash
[certs] Using certificateDir folder "/etc/kubernetes/pki"
```
- 인증서 저장 디렉토리를 `/etc/kubernetes/pki`로 설정

```bash
[certs] Generating "ca" certificate and key
```
- CA(Certificate Authority) 인증서 및 키 생성 시작

```bash
[certs] Generating "apiserver" certificate and key
```
- API 서버 인증서 및 키 생성

```bash
[certs] apiserver serving cert is signed for DNS names [k8s-master k8smaster kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.9.19]
```
- API 서버 인증서에 다음 DNS 이름 및 IP 포함:
  - DNS: k8s-master, k8smaster, kubernetes, kubernetes.default 등
  - IP: 10.96.0.1, 10.0.9.19

```bash
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
```
- 추가 인증서 생성:
  - API 서버-kubelet 클라이언트 인증서
  - 프론트 프록시 CA 및 클라이언트 인증서
  - etcd CA 및 서버 인증서

```bash
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [10.0.9.19 127.0.0.1 ::1]
```
- etcd 서버 인증서에 다음 DNS 및 IP 포함:
  - DNS: k8s-master, localhost
  - IP: 10.0.9.19, 127.0.0.1, ::1

```bash
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
```
- Kubeconfig 파일 생성:
  - admin.conf
  - kubelet.conf
  - controller-manager.conf
  - scheduler.conf

```bash
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
```
- 컨트롤 플레인 구성 요소의 정적 Pod 매니페스트 생성:
  - etcd
  - kube-apiserver
  - kube-controller-manager
  - kube-scheduler

```bash
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
```
- kubelet 시작 준비:
  - 환경 파일 작성
  - 구성 파일 작성
  - kubelet 서비스 시작

```bash
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
```
- 컨트롤 플레인 구성 요소가 완전히 시작될 때까지 최대 4분 대기

```bash
[apiclient] All control plane components are healthy after 8.003978 seconds
```
- 모든 컨트롤 플레인 구성 요소가 8초 후 정상 상태 확인

```bash
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
```
- 클러스터 구성 정보를 ConfigMap에 저장

```bash
[upload-certs] Skipping phase. Please see --upload-certs
```
- 인증서 업로드 단계 건너뜀

```bash
[mark-control-plane] Marking the node k8s-master as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
```
- k8s-master 노드에 레이블 및 테인트 추가:
  - 컨트롤 플레인 노드로 표시
  - 외부 로드 밸런서에서 제외
  - 기본적으로 워크로드 스케줄링 금지

```bash
[bootstrap-token] Using token: vd1ppv.2g3q2tm6nxfmsnir
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
```
- 부트스트랩 토큰 생성 및 RBAC 규칙 구성

```bash
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
```
- 부트스트랩 토큰에 대한 다양한 RBAC 규칙 구성

```bash
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
```
- 클러스터 정보 ConfigMap 생성

```bash
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
```
- kubelet 클라이언트 인증서 회전 설정

```bash
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy
```
- 필수 애드온 설치:
  - CoreDNS
  - kube-proxy

### 4. 클러스터 노드 추가 방법

#### 컨트롤 플레인 노드 추가
```bash
kubeadm join k8smaster:6443 \
  --token vd1ppv.2g3q2tm6nxfmsnir \
  --discovery-token-ca-cert-hash sha256:c5a0aa585806ba0a36d61b34b4881aeecdddf3e451570878045de01827333a25 \
  --control-plane
```

#### 워커 노드 추가
```bash
kubeadm join k8smaster:6443 \
  --token vd1ppv.2g3q2tm6nxfmsnir \
  --discovery-token-ca-cert-hash sha256:c5a0aa585806ba0a36d61b34b4881aeecdddf3e451570878045de01827333a25
```

### 5. 주의사항 및 권장사항

- Pod 네트워크 애드온을 반드시 설치해야 합니다.
- 컨테이너 런타임의 샌드박스 이미지는 `registry.k8s.io/pause:3.9` 버전 사용을 권장합니다.
- 보안을 위해 토큰과 인증서 해시는 주기적으로 갱신하는 것이 좋습니다.

### 6. 사용자 환경 설정

#### 일반 사용자 kubectl 설정
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 7. 추가 참고 사항

- 공식 Kubernetes 문서: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/
- Pod 네트워크 애드온 선택: https://kubernetes.io/docs/concepts/cluster-administration/addons/

### 8. 문제 해결

- 클러스터 초기화 중 문제 발생 시 `kubeadm reset` 명령어로 초기화 상태를 초기화할 수 있습니다.
- 로그는 `/var/log/kubernetes/` 디렉토리에서 확인 가능합니다.
