# 목표

1. EC2애 kubernetes 클러스터 구축

https://bydawn25.tistory.com/78

VPC 구성

EC2 생성
개수 : 4개
Type: t3.large
OS : ubuntu 22.04
Role : SSM


## 1. EC2 초기 세팅

### 1-1. AWS CLI 설치
sudo -i
apt update && apt install -y awscli


### 1-2. hostname 설정
sudo -i
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600") && INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id) && REGION=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/region) && NAME_TAG=$(aws ec2 describe-tags --region $REGION --filters "Name=resource-id,Values=$INSTANCE_ID" "Name=key,Values=Name" --query "Tags[0].Value" --output text)
hostnamectl set-hostname "$NAME_TAG"

### 1-3. swap off
sudo -i
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

/etc/fstab 파일에서 "swap"이 포함된 줄을 찾아 그 앞에 '#'을 붙여 주석 처리함으로써 재부팅 후에도 스왑이 비활성화된 상태로 유지하는 명령어



### 1-4. 커널 모듈 설정 파일 생성 
sudo tee /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF

### 커널 모듈 로드
sudo modprobe overlay && sudo modprobe br_netfilter

### 커널 모듈 로드 확인
lsmod | grep -E 'br_netfilter|overlay'

### 1-5. 커널 파라미터 설정
sudo tee /etc/sysctl.d/k8s.conf << 'EOF'
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

### 커널 파라미터 설정 적용
sudo sysctl --system



### 1-6 containerd 설치
sudo -i
wget https://github.com/containerd/containerd/releases/download/v1.6.9/containerd-1.6.9-linux-amd64.tar.gz && sudo tar Cxzvf /usr/local containerd-1.6.9-linux-amd64.tar.gz

- service 등록
sudo mkdir -p /usr/local/lib/systemd/system && sudo wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -O /usr/local/lib/systemd/system/containerd.service && sudo systemctl daemon-reload && sudo systemctl enable --now containerd && systemctl status containerd


### 1-7 runc 설치
wget https://github.com/opencontainers/runc/releases/download/v1.1.3/runc.amd64 && sudo install -m 755 runc.amd64 /usr/local/sbin/runc

### 1-8 CNI plugin 설치
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz && sudo mkdir -p /opt/cni/bin && sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz

### 1-9 containerd를 CRI런타임으로 사용하기 위한 설정 변경
sudo mkdir -p /etc/containerd && sudo containerd config default | sudo tee /etc/containerd/config.toml && sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml && sudo systemctl restart containerd


sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg && echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list && sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl && sudo apt-mark hold kubelet kubeadm kubectl
