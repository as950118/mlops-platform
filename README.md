# MLOps Platform

비전 AI 모델 개발을 위한 엔드투엔드 MLOps 플랫폼 프로젝트입니다.

## 📋 프로젝트 개요

본 프로젝트는 ML 엔지니어가 격리된 개발 환경에서 모델을 개발하고, 라벨링된 데이터로 지속적으로 모델을 개선하며, 안전하게 배포할 수 있는 MLOps 플랫폼을 설계하고 구현하는 것을 목표로 합니다.

### 문제 상황

- 각 엔지니어가 로컬 환경에서 개발하여 환경 불일치와 재현성 문제 발생
- 학습된 모델을 수동으로 공유하는 방식으로 인한 비효율성
- 새로운 데이터 생성 시 모델 학습 반영 과정이 수동적
- 모델 배포 시 성능 검증 없이 교체하여 데이터 품질 저하 발생

### 해결 목표

- 격리된 GPU 개발 환경 제공
- 자동화된 모델 학습 파이프라인 구축
- 안전한 모델 배포 프로세스 구현
- 분산 학습 지원

## 🎯 주요 기능

### 1. 모델 개발 환경

- **격리된 GPU 개발 환경 할당**
  - ML 엔지니어가 GPU 자원(종류, 수량)을 요청하면 격리된 개발 환경 할당
  - 엔지니어의 IDE에서 원격으로 해당 환경에 접속하여 개발
  - 컨테이너 기반으로 제공되며, 엔지니어별로 독립적인 환경 보장

### 2. 모델 학습 파이프라인

- **실험 추적 및 버전 관리**
  - 학습된 모델의 메트릭, 하이퍼파라미터, 아티팩트 추적 및 버전 관리
  - 모델은 단계별(실험 → 검증 → 운영)로 관리
  - 검증을 통과한 모델만 운영 환경에 배포

- **자동 재학습**
  - 새로운 라벨링 데이터가 일정 수준 누적되면 자동으로 모델 재학습 트리거
  - 재학습 시 기존 모델의 지식을 유지하면서 새로운 데이터를 학습 (전이학습)

### 3. 모델 서빙

- **다양한 서빙 형태 지원**
  - **Batch 서빙**: 라벨링 데이터의 전처리 과정에서 전체 데이터에 대한 Pseudo labeling 수행
  - **One-by-One 서빙**: 라벨링 도구에서 서빙 API를 호출하여 예측 결과를 미리 채우고, 라벨러가 검수/수정

- **안전한 배포**
  - 새 모델 배포 시 일부 트래픽만 새 모델로 전환하여 성능 검증 (Canary 배포)
  - 문제가 없으면 전체 트래픽을 전환

### 4. 분산 학습

- **분산 학습 지원**
  - ML 엔지니어가 작성한 단일 GPU 학습 코드를 최소한의 수정으로 분산 학습 환경에서 실행 가능
  - 분산 학습 작업의 스케줄링 및 모니터링 지원

## 📐 설계 산출물

### 종합 설계 문서

**[MLOps Platform 설계 문서](./docs/MLOps-Platform-Design.md)** ✅
- 전체 시스템 아키텍처 및 데이터 흐름
- Sequence Diagram (모델 개발, 재학습, 배포)
- 기술 스택 선정 및 비교 분석
- 예상 이슈 및 해결 방안
- 구현 계획

### 상세 문서

1. **[System Architecture](./docs/architecture/system-architecture.md)** ✅
2. **[Development Environment](./docs/architecture/development-environment.md)** ✅
3. **[GPU Allocation](./docs/architecture/gpu-allocation.md)** ✅
4. **[Implementation Plan](./docs/architecture/implementation-plan.md)** ✅
5. **[Tech Stack Selection](./docs/architecture/tech-stack-selection.md)** ✅
6. **[Sequence Diagrams](./docs/sequence/model-lifecycle.md)** ✅
7. **[Design Decisions](./docs/design-decisions.md)** ✅

## 🛠 기술 스택

- **컨테이너 오케스트레이션**: Kubernetes
- **MLOps 플랫폼**: MLflow (실험 추적 및 모델 레지스트리), Kubeflow Pipelines (워크플로우 오케스트레이션)
- **모델 서빙**: KServe (Batch 및 Real-time 서빙)
- **분산 학습**: PyTorch DDP / Horovod
- **데이터 버전 관리**: DVC (Data Version Control)
- **모니터링**: Prometheus, Grafana, ELK Stack
- **CI/CD**: GitHub Actions, ArgoCD

자세한 기술 스택 선택 이유는 [Design Decisions](./docs/design-decisions.md) 문서를 참고하세요.

## 📁 프로젝트 구조

```
mlops-platform/
├── README.md
├── docs/
│   ├── architecture/
│   │   ├── system-architecture.md
│   │   ├── development-environment.md
│   │   ├── gpu-allocation.md
│   │   ├── implementation-plan.md
│   │   ├── tech-stack-selection.md
│   │   └── authentication-authorization.md
│   ├── sequence/
│   │   └── model-lifecycle.md
│   └── design-decisions.md
├── infrastructure/
│   ├── kubernetes/
│   └── terraform/
└── examples/
    └── sample-training-code/
```

## 🚀 시작하기

### 사전 요구사항

- Kubernetes 클러스터 (로컬: minikube, kind 등)
- Docker
- kubectl

### 설치 방법

```bash
# 저장소 클론
git clone <repository-url>
cd mlops-platform

# Kubernetes 리소스 배포
kubectl apply -f infrastructure/kubernetes/
```

## 📝 개발 로드맵

### 설계 단계 (완료)
- [x] 시스템 아키텍처 설계
- [x] 시퀀스 다이어그램 작성
- [x] 기술 스택 비교 분석
- [x] 예상 이슈 및 해결 방안 문서화
- [x] GPU 할당 및 분할 전략 수립
- [x] 구현 플랜 수립

### Phase 1: 기본 인프라 (진행 중)
- [ ] 네임스페이스 및 ResourceQuota 설정
- [ ] LimitRange 설정
- [ ] DevEnvironment CRD 정의
- [ ] 기본 Kubernetes 리소스 배포

### Phase 2: Controller 구현
- [ ] DevEnvironment Controller 구현
- [ ] Port Manager 구현
- [ ] Resource Scheduler 구현
- [ ] TTL 기반 자동 정리 기능

### Phase 3: API 서버
- [ ] REST API 서버 구현
- [ ] 환경 생성/조회/삭제 API
- [ ] 인증 및 권한 관리

### Phase 4: 통합 및 테스트
- [ ] MLflow 및 Kubeflow Pipelines 설정
- [ ] 모델 학습 파이프라인 구현
- [ ] 자동 재학습 트리거 구현
- [ ] 모델 서빙 인프라 구축 (KServe)
- [ ] Canary 배포 구현
- [ ] 분산 학습 지원 구현

### Phase 5: 모니터링 및 운영
- [ ] 모니터링 및 로깅 시스템 구축
- [ ] 대시보드 구성
- [ ] 알림 시스템 구축

자세한 구현 계획은 [Implementation Plan](./docs/architecture/implementation-plan.md) 문서를 참고하세요.

## 🤝 기여하기

이 프로젝트는 학습 및 포트폴리오 목적으로 진행되는 토이 프로젝트입니다.

## 📄 라이선스

MIT License
