# Kubernetes 클러스터 초기화 문제 해결

## 발생한 에러

```
[ERROR CRI]: container runtime is not running: output: time="2025-03-14T13:14:20Z" level=fatal msg="validate service connection: validate CRI v1 runtime API for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService"
```

## 문제 원인

1. **CRI 플러그인 로드 실패**: containerd 로그에서 `"failed to load plugin io.containerd.grpc.v1.cri"` 에러 발생
2. **containerd 설정 문제**: systemd_cgroup 설정이 일관되지 않고 sandbox 이미지 버전 불일치
3. **설정 파일 충돌**: 여러 번 설정 변경으로 인한 config.toml 파일의 손상 가능성

## 진단 방법

1. **containerd 상태 확인**: `systemctl status containerd`로 로그 메시지 확인
2. **CRI 플러그인 확인**: `ctr --address /run/containerd/containerd.sock plugin ls | grep cri`로 CRI 플러그인 상태 확인
3. **소켓 파일 확인**: `ls -la /run/containerd/containerd.sock`으로 소켓 파일 존재 여부 확인

## 해결 방법

1. **containerd 중지 및 설정 백업**:
   ```bash
   sudo systemctl stop containerd
   sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
   ```

2. **기본 설정 생성 및 필요한 설정 적용**:
   ```bash
   sudo mkdir -p /etc/containerd
   sudo containerd config default | sudo tee /etc/containerd/config.toml
   sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
   ```

3. **containerd 재시작 및 상태 확인**:
   ```bash
   sudo systemctl restart containerd
   sudo systemctl status containerd
   ```

## 기술적 설명

1. **CRI(Container Runtime Interface)**: Kubernetes가 컨테이너 런타임과 통신하는 표준 인터페이스입니다. kubeadm은 이 인터페이스를 통해 containerd와 통신합니다.

2. **containerd 설정 파일 문제**: 
   - 여러 번 수정으로 인해 설정 파일이 손상되었을 수 있습니다.
   - `SystemdCgroup` 설정과 같은 핵심 설정이 일관되지 않을 수 있습니다.

3. **해결 접근 방식**:
   - 기존 설정 파일을 백업하고 완전히 새로운 기본 설정으로 시작합니다.
   - 기본 설정 생성 후 필요한 변경사항만 적용합니다.
   - 이렇게 하면 설정 파일 문법 오류나 누락된 섹션 문제가 해결됩니다.

## 향후 문제 방지 방법

1. 설정 변경 시 항상 백업 파일 생성
2. 설정 변경 후 containerd 상태 확인
3. 설정 변경 사항 문서화
4. containerd 버전과 호환되는 Kubernetes 버전 확인

이 방법으로 설정 파일을 깨끗하게 재설정함으로써 containerd의 CRI 플러그인이 올바르게 로드되어 쿠버네티스 클러스터를 성공적으로 초기화할 수 있게 되었습니다.
