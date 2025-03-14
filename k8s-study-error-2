# kubectl 워커 노드 설정 문제 분석

## 발생한 에러

```
E0314 13:29:35.868907    4706 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

## 문제 원인

1. **아키텍처 구조적 요인**: 워커 노드에는 API 서버가 실행되지 않음
2. **설정 파일 부재**: `/etc/kubernetes/admin.conf` 파일이 워커 노드에 존재하지 않음
3. **참조 구성**: 환경 변수(KUBECONFIG)가 존재하지 않는 파일을 참조
4. **권한 분리**: 쿠버네티스 아키텍처는 의도적으로 컨트롤 플레인과 워커 노드의 역할을 분리함

## 기술적 설명

### 쿠버네티스 클러스터 아키텍처
1. **컨트롤 플레인 (마스터 노드)**:
   - API 서버, 스케줄러, 컨트롤러 관리자, etcd 등 실행
   - 클러스터 상태 관리 및 제어 기능 담당
   - kubectl 명령에 필요한 인증 정보와 설정 파일 보유

2. **워커 노드**:
   - kubelet, kube-proxy, 컨테이너 런타임(containerd) 실행
   - 컨테이너 실행 환경 제공
   - API 서버에 접근할 수 있는 인증 정보가 제한적

### kubectl 작동 방식
1. kubectl은 kubeconfig 파일을 사용하여 API 서버에 연결
2. 기본적으로 `$HOME/.kube/config` 또는 `KUBECONFIG` 환경 변수 경로의 파일 참조
3. 워커 노드에는 이 설정 파일이 기본적으로 존재하지 않음

## 해결 방안

### 권장 방법
- **마스터 노드에서만 kubectl 사용** (표준 방식)
- 마스터 노드에서 `export KUBECONFIG=/etc/kubernetes/admin.conf` 설정

## 디자인 원칙
이 구조는 보안 및 책임 분리를 위한 의도적 설계입니다:
- **권한 분리**: 워커 노드는 제한된 권한만 가짐
- **공격 표면 감소**: 클러스터 관리 기능을 컨트롤 플레인에만 국한
- **운영 명확성**: 관리 작업은 컨트롤 플레인에서만 수행되어 일관성 유지

이런 설계로 인해 워커 노드에서 kubectl 설정을 시도할 때 "connection refused" 에러가 발생하는 것은 예상된 동작이며, 정상적인 쿠버네티스 설계의 일부입니다.
