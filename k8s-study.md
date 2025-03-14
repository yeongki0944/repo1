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
