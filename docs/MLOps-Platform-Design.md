# MLOps Platform 설계 문서

## 목차

1. [개요](#개요)
2. [System Architecture](#system-architecture)
3. [Sequence Diagrams](#sequence-diagrams)
4. [기술 스택 선정](#기술-스택-선정)
5. [예상 이슈 및 해결 방안](#예상-이슈-및-해결-방안)
6. [구현 계획](#구현-계획)

---

## 개요

### 프로젝트 목표

ML 엔지니어가 격리된 개발 환경에서 모델을 개발하고, 라벨링된 데이터로 지속적으로 모델을 개선하며, 안전하게 배포할 수 있는 MLOps 플랫폼을 설계합니다.

### 핵심 요구사항

1. **모델 개발 환경**: 격리된 GPU 개발 환경 제공
2. **모델 학습 파이프라인**: 자동화된 실험 추적 및 재학습
3. **모델 서빙**: Batch 및 Real-time 서빙 지원
4. **안전한 배포**: Canary 배포를 통한 점진적 배포
5. **분산 학습**: 대용량 데이터셋 분산 학습 지원
6. **멀티클러스터**: 여러 Kubernetes 클러스터 관리
7. **제로 트러스트**: 네트워크 기반 신뢰 없이 모든 트래픽 검증

---

## System Architecture

### 전체 시스템 아키텍처

```mermaid
graph TB
    subgraph "Multi-Cluster Management"
        FLEET[Fleet Manager]
        ARGO[ArgoCD<br/>GitOps]
    end

    subgraph "API & Gateway Layer"
        API[API Server<br/>REST API]
        ISTIO_GW[Istio Gateway<br/>Service Mesh]
    end

    subgraph "Identity & Security"
        KEYCLOAK[Keycloak<br/>Identity Provider]
        VAULT[Vault<br/>Secrets Management]
        CILIUM[Cilium CNI<br/>Network Policy]
    end

    subgraph "Development Environment"
        CONTROLLER[Dev Environment<br/>Controller]
        DEV_NS[Namespace: mlops-dev<br/>Pod 단위 격리]
        GPU[GPU Resources<br/>MIG/Time-Slicing/Full GPU]
    end

    subgraph "MLOps Platform"
        MLFLOW[MLflow<br/>Experiment Tracking]
        KUBEFLOW[Kubeflow Pipelines<br/>Workflow Orchestration]
        MODEL_REG[Model Registry<br/>MLflow]
    end

    subgraph "Training Infrastructure"
        TRAIN_NS[Namespace: mlops-training]
        DIST_TRAIN[Distributed Training<br/>PyTorch DDP]
    end

    subgraph "Model Serving"
        SERVING_NS[Namespace: mlops-serving]
        KSERVE[KServe<br/>Batch & Real-time]
        CANARY[Canary Deployment<br/>Istio Traffic Splitting]
    end

    subgraph "Data Management"
        MINIO[MinIO<br/>Object Storage]
        DVC[DVC<br/>Data Versioning]
        LABELING[Labeling Service]
    end

    subgraph "Observability"
        OTEL[OpenTelemetry<br/>Collector]
        SIGNOZ[SigNoz<br/>Logs + Metrics + Traces]
    end

    subgraph "Shared Services"
        POSTGRES[PostgreSQL<br/>MLflow Backend]
        REDIS[Redis<br/>Cache]
    end

    FLEET --> ISTIO_GW
    ARGO --> ISTIO_GW
    API --> CONTROLLER
    CONTROLLER --> DEV_NS
    DEV_NS --> GPU
    ISTIO_GW --> KEYCLOAK
    ISTIO_GW --> VAULT
    ISTIO_GW --> CILIUM
    DEV_NS --> MLFLOW
    KUBEFLOW --> MLFLOW
    KUBEFLOW --> TRAIN_NS
    TRAIN_NS --> DIST_TRAIN
    DIST_TRAIN --> GPU
    MODEL_REG --> KSERVE
    KSERVE --> CANARY
    LABELING --> MINIO
    MINIO --> DVC
    ISTIO_GW --> OTEL
    DIST_TRAIN --> OTEL
    KSERVE --> OTEL
    OTEL --> SIGNOZ
    MLFLOW --> POSTGRES
    KSERVE --> REDIS
```

### 데이터 흐름

```mermaid
flowchart LR
    subgraph "Data Flow"
        A[New Labeled Data] --> B[MinIO Data Lake]
        B --> C[DVC Versioning]
        C --> D{Threshold<br/>Reached?}
        D -->|Yes| E[Kubeflow Pipeline<br/>Trigger]
        D -->|No| F[Wait]
        E --> G[Training Job]
        G --> H[MLflow Tracking]
        H --> I[Model Validation]
        I --> J{Validation<br/>Pass?}
        J -->|Yes| K[Model Registry<br/>Staging]
        J -->|No| L[Reject]
        K --> M[KServe Canary<br/>10% Traffic]
        M --> N{Performance<br/>OK?}
        N -->|Yes| O[Full Deployment<br/>100% Traffic]
        N -->|No| P[Rollback]
        O --> Q[Production Serving]
    end
```

### 네임스페이스 구조

```mermaid
graph TB
    subgraph "Kubernetes Cluster"
        subgraph "mlops-dev"
            DEV_POD1[Pod: dev-env-1<br/>GPU: 다양한 방식 지원]
            DEV_POD2[Pod: dev-env-2<br/>GPU: 다양한 방식 지원]
            DEV_SVC1[Service: NodePort<br/>30001-30003]
        end
        
        subgraph "mlops-training"
            TRAIN_JOB1[Job: training-1<br/>GPU: 요구사항에 따라 할당]
            TRAIN_JOB2[Job: training-2<br/>GPU: 요구사항에 따라 할당]
        end
        
        subgraph "mlops-serving"
            SERVING1[KServe: model-v1<br/>Canary 10%]
            SERVING2[KServe: model-v2<br/>Canary 90%]
        end
        
        subgraph "mlops-platform"
            MLFLOW_SVC[MLflow Server]
            MINIO_SVC[MinIO]
        end
    end
```

### 구성요소 상세 설명

#### 1. Development Environment Manager
- **역할**: ML 엔지니어의 격리된 GPU 개발 환경 할당 및 관리
- **기술**: Kubernetes Custom Controller (Operator Pattern)
- **격리 전략**: 
  - 네임스페이스: 목적별 분리 (mlops-dev, mlops-training, mlops-serving)
  - Pod 단위 격리: 공통 네임스페이스 내 Pod 레벨 격리
  - 유연한 GPU 할당: 요구사항에 따라 다양한 GPU 할당 방식 지원
- **GPU 할당 방식**:
  - **MIG (Multi-Instance GPU)**: A100/H100의 하드웨어 레벨 분할 (격리 우수)
  - **Time-Slicing**: 시간 분할 방식으로 여러 Pod이 GPU 공유 (비용 효율)
  - **전체 GPU 할당**: 하나의 Pod에 전체 GPU 할당 (성능 우선)
  - 기타: CUDA MPS 등 다양한 방식 지원
- **선택 이유**:
  - 확장성: 수백 명의 개발자 지원 가능
  - 리소스 효율: 네임스페이스 수 최소화
  - 격리 보장: Pod 레벨에서 충분한 격리
  - 유연성: 다양한 GPU 할당 방식으로 요구사항에 맞춤

#### 2. Istio Service Mesh & API Gateway
- **역할**: 서비스 메시, API Gateway, 제로 트러스트 네트워킹
- **기술**: Istio
- **주요 기능**:
  - mTLS: 모든 서비스 간 암호화 통신
  - AuthorizationPolicy: 세밀한 접근 제어
  - Gateway: Ingress/Egress 게이트웨이
  - VirtualService: 트래픽 라우팅
  - 자동 모니터링: Envoy가 자동으로 메트릭/트레이스 생성
- **선택 이유**:
  - 서비스 메시와 API Gateway 통합
  - 제로 트러스트 구현 용이
  - OpenTelemetry와 완벽 통합
  - 멀티클러스터 지원

#### 3. MLflow
- **역할**: 실험 추적, 메트릭/파라미터/아티팩트 저장, 모델 레지스트리
- **기술**: MLflow
- **선택 이유**:
  - 실험 추적과 모델 레지스트리 통합 제공
  - 다양한 ML 프레임워크 지원
  - 오픈소스로 자체 호스팅 가능
  - REST API를 통한 쉬운 통합

#### 4. Kubeflow Pipelines
- **역할**: 모델 학습 파이프라인 오케스트레이션
- **기술**: Kubeflow Pipelines
- **선택 이유**:
  - Kubernetes 네이티브 워크플로우 오케스트레이션
  - 재사용 가능한 컴포넌트 기반 파이프라인
  - 자동 재학습 트리거 및 스케줄링 지원
  - 파이프라인 버전 관리 및 재현성 보장

#### 5. KServe
- **역할**: 모델 서빙 (Batch 및 Real-time)
- **기술**: KServe
- **선택 이유**:
  - Kubernetes 네이티브 서빙 솔루션
  - Batch와 Real-time 서빙 모두 지원
  - 자동 스케일링 및 A/B 테스팅 지원
  - Istio와의 통합을 통한 Canary 배포

#### 6. OpenTelemetry + SigNoz
- **역할**: 통합 관찰 가능성 (로그, 메트릭, 트레이스)
- **기술**: OpenTelemetry Collector + SigNoz
- **선택 이유**:
  - 통합 표준: 하나의 표준으로 모든 관찰 가능성 데이터 수집
  - 올인원 솔루션: SigNoz로 로그/메트릭/트레이스 통합 관리
  - OpenTelemetry 네이티브: Istio와 완벽 통합
  - 고성능: ClickHouse 기반 저장소

#### 7. Cilium CNI
- **역할**: CNI 및 네트워크 레벨 정책 (L3/L4)
- **기술**: Cilium
- **선택 이유**:
  - eBPF 기반 고성능 네트워킹
  - 네트워크 레벨 정책 (L3/L4)
  - 멀티클러스터 지원
  - Istio와 함께 사용 가능 (역할 분리)

**Istio와 Cilium의 역할 분리**:
- **Cilium**: CNI로 네트워크 레벨 정책 (L3/L4), Pod 간 기본 네트워크 격리
- **Istio (Envoy)**: 서비스 메시로 애플리케이션 레벨 정책 (L7), mTLS, 트래픽 관리
- **함께 사용**: Cilium이 네트워크 레벨에서 기본 보안 제공, Istio가 L7 레벨에서 세밀한 제어

#### 8. Keycloak + Vault
- **역할**: Identity Provider 및 Secrets Management
- **기술**: Keycloak, Vault
- **선택 이유**:
  - 완전한 오픈소스
  - OIDC, OAuth2 지원
  - 중앙화된 시크릿 관리
  - Kubernetes 통합

---

## Sequence Diagrams

### 1. ML 엔지니어의 모델 개발 및 학습 흐름

```mermaid
sequenceDiagram
    participant ML as ML Engineer
    participant API as API Server
    participant Controller as Dev Env Controller
    participant K8s as Kubernetes
    participant Pod as GPU Pod (MIG)
    participant MLflow as MLflow
    participant Pipeline as Kubeflow Pipeline
    participant DataLake as MinIO

    Note over ML,DataLake: 1. 개발 환경 요청
    ML->>API: POST /api/v1/environments<br/>{engineer_id, gpu_type, gpu_allocation}
    API->>Controller: Create DevEnvironment CR
    Controller->>K8s: Create Pod in mlops-dev
    Controller->>K8s: Create Service (NodePort)
    Controller->>K8s: Create PVC
    K8s->>Pod: Schedule to GPU Node
    Pod->>Pod: Initialize Container
    Pod-->>Controller: Pod Ready
    Controller-->>API: Environment Ready<br/>{node_port: 30001, access_url}
    API-->>ML: Environment Details

    Note over ML,DataLake: 2. IDE 접속 및 개발
    ML->>Pod: Connect via NodePort<br/>http://node:30001
    Pod-->>ML: VS Code Server Ready
    ML->>Pod: Write Training Code

    Note over ML,DataLake: 3. 데이터 준비
    ML->>DataLake: Request Dataset
    DataLake-->>ML: Dataset URL
    ML->>Pod: Download Dataset

    Note over ML,DataLake: 4. 실험 실행 및 추적
    ML->>Pod: Run Training Script
    Pod->>MLflow: Log Metrics, Params, Artifacts
    MLflow-->>Pod: Tracking Confirmed
    Pod->>Pod: Training Execution
    Pod->>MLflow: Log Model Checkpoints
    Pod->>MLflow: Log Final Metrics
    Pod-->>ML: Training Complete

    Note over ML,DataLake: 5. 모델 등록
    ML->>MLflow: Register Model (Experiment)
    MLflow-->>ML: Model Registered

    Note over ML,DataLake: 6. 분산 학습 (선택적)
    alt 분산 학습 필요 시
        ML->>Pipeline: Submit Distributed Training
        Pipeline->>K8s: Create Multi-GPU Job
        K8s->>K8s: Allocate Multiple GPU Resources
        K8s->>K8s: Initialize DDP
        K8s->>K8s: Distributed Training
        K8s->>MLflow: Log Distributed Metrics
        K8s-->>Pipeline: Training Complete
        Pipeline-->>ML: Job Complete
    end
```

### 2. 새로운 데이터 누적 → 재학습 → 배포까지의 흐름

```mermaid
sequenceDiagram
    participant Labeler as Labeling Team
    participant LabelSvc as Labeling Service
    participant DataLake as MinIO
    participant DVC as DVC
    participant Monitor as Data Monitor
    participant Pipeline as Kubeflow Pipeline
    participant MLflow as MLflow
    participant Validator as Model Validator
    participant Registry as Model Registry
    participant KServe as KServe
    participant Istio as Istio Gateway
    participant Traffic as Production Traffic

    Note over Labeler,Traffic: 1. 새로운 데이터 라벨링
    Labeler->>LabelSvc: Label New Data
    LabelSvc->>DataLake: Save Labeled Data
    DataLake->>DVC: Version Dataset
    DVC-->>LabelSvc: Dataset Versioned
    LabelSvc-->>Labeler: Labeling Complete

    Note over Labeler,Traffic: 2. 데이터 누적 모니터링
    DataLake->>Monitor: Update Data Count
    Monitor->>Monitor: Check Threshold
    alt Threshold Reached
        Monitor->>Pipeline: Trigger Retraining Pipeline
    else Threshold Not Reached
        Monitor->>Monitor: Continue Monitoring
    end

    Note over Labeler,Traffic: 3. 자동 재학습 트리거
    Pipeline->>DataLake: Fetch Latest Dataset
    DataLake-->>Pipeline: Dataset URL
    Pipeline->>Registry: Load Base Model (Production)
    Registry-->>Pipeline: Model Artifacts
    Pipeline->>MLflow: Start New Experiment
    MLflow-->>Pipeline: Experiment ID

    Note over Labeler,Traffic: 4. 전이학습 수행
    Pipeline->>Pipeline: Initialize Model with<br/>Base Model Weights
    Pipeline->>Pipeline: Fine-tune on New Data
    Pipeline->>MLflow: Log Training Metrics
    Pipeline->>MLflow: Log Hyperparameters
    Pipeline->>MLflow: Save Model Checkpoints
    Pipeline-->>Pipeline: Training Complete

    Note over Labeler,Traffic: 5. 모델 검증
    Pipeline->>Validator: Request Model Validation
    Validator->>DataLake: Load Validation Dataset
    Validator->>MLflow: Load Trained Model
    Validator->>Validator: Run Validation Tests
    Validator->>Validator: Calculate Metrics
    alt Validation Passed
        Validator->>Registry: Promote to Staging
        Registry->>Registry: Update Model Stage
        Registry-->>Pipeline: Model in Staging
    else Validation Failed
        Validator->>Registry: Reject Model
        Registry-->>Pipeline: Model Rejected
        Pipeline->>MLflow: Log Failure Reason
    end

    Note over Labeler,Traffic: 6. Canary 배포 (검증 통과 시)
    alt Model in Staging
        Pipeline->>KServe: Deploy Model (Canary)
        KServe->>Istio: Configure Traffic Split (10%)
        Istio->>Traffic: Route 10% to New Model
        Istio->>Traffic: Route 90% to Old Model
        
        Note over Istio,Traffic: 7. 성능 모니터링
        Traffic->>Istio: Request Traffic
        Istio->>KServe: Forward to Model
        KServe-->>Istio: Prediction Results
        Istio->>Monitor: Collect Performance Metrics
        Monitor->>Monitor: Compare Old vs New Model
        
        alt Performance OK
            Monitor->>Istio: Approve Full Deployment
            Istio->>Istio: Update Traffic Split (100%)
            Istio->>Traffic: Route 100% to New Model
            Istio->>Registry: Promote to Production
            Registry->>Registry: Update Model Stage
            Registry-->>KServe: Model in Production
        else Performance Degraded
            Monitor->>Istio: Trigger Rollback
            Istio->>Istio: Revert Traffic Split (0%)
            Istio->>Traffic: Route 100% to Old Model
            Istio->>Registry: Keep Old Model in Production
            Registry-->>Pipeline: Rollback Complete
        end
    end
```

### 3. 모델 서빙 흐름 (Batch 및 One-by-One)

```mermaid
sequenceDiagram
    participant Client as Client Application
    participant LabelTool as Labeling Tool
    participant Istio as Istio Gateway
    participant KServe as KServe
    participant Batch as Batch Serving
    participant Realtime as Real-time Serving
    participant Model as Model Instance

    Note over Client,Model: One-by-One 서빙 (Real-time)
    LabelTool->>Istio: POST /predict (Single Request)
    Istio->>Istio: JWT Authentication
    Istio->>Realtime: Forward Request
    Realtime->>Model: Load Model & Predict
    Model-->>Realtime: Prediction Result
    Realtime-->>Istio: Return Result
    Istio-->>LabelTool: Prediction (Pre-filled)
    LabelTool->>LabelTool: Labeler Review/Edit
    LabelTool->>LabelTool: Save Final Label

    Note over Client,Model: Batch 서빙
    Client->>Istio: POST /batch-predict (Dataset)
    Istio->>Istio: JWT Authentication
    Istio->>Batch: Submit Batch Job
    Batch->>Model: Load Model
    Batch->>Batch: Process Dataset in Batches
    Batch->>Model: Batch Predictions
    Model-->>Batch: Batch Results
    Batch->>Batch: Aggregate Results
    Batch-->>Istio: Complete Batch Results
    Istio-->>Client: Batch Predictions (Pseudo Labels)
```

---

## 기술 스택 선정

### 최종 기술 스택

| 카테고리 | 기술 | 선택 이유 |
|---------|------|----------|
| **멀티클러스터 관리** | Fleet + ArgoCD | 클러스터 수준 관리 + GitOps 배포 |
| **서비스 메시 & API Gateway** | Istio (Envoy) | 서비스 메시와 API Gateway 통합, L7 제어, 제로 트러스트, 자동 모니터링 |
| **네트워크 보안** | Cilium CNI | CNI 및 네트워크 레벨 정책 (L3/L4), eBPF 기반 고성능 |
| **Identity & Access** | Keycloak + Vault | OIDC/OAuth2 지원, 중앙화된 시크릿 관리 |
| **관찰 가능성** | OpenTelemetry + SigNoz | 통합 표준, 올인원 솔루션, 고성능 |
| **MLOps 플랫폼** | MLflow + Kubeflow + KServe | 실험 추적, 파이프라인, 서빙 통합 |
| **데이터** | MinIO + PostgreSQL + Redis | S3 호환, 관계형 DB, 캐싱 |

**Cilium과 Istio의 역할 분리**:
- **Cilium**: CNI로 네트워크 레벨 정책 (L3/L4), Pod 간 기본 네트워크 격리
- **Istio (Envoy)**: 서비스 메시로 애플리케이션 레벨 정책 (L7), mTLS, 트래픽 관리
- **함께 사용**: 다층 방어 전략 - Cilium이 네트워크 레벨에서 기본 보안 제공, Istio가 L7에서 세밀한 제어

### 기술 스택 비교

#### 서비스 메시 & API Gateway

| 항목 | Istio | Kong | Traefik |
|------|-------|------|---------|
| **서비스 메시** | ⭐⭐⭐⭐⭐ | ❌ | ❌ |
| **API Gateway** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **제로 트러스트** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **모니터링 통합** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **멀티클러스터** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **선택** | ✅ | ❌ | ❌ |

**선택 이유**: Istio는 서비스 메시와 API Gateway를 통합 제공하며, OpenTelemetry와 완벽 통합되어 자동 모니터링이 가능합니다.

#### 관찰 가능성

| 항목 | OpenTelemetry + SigNoz | Prometheus + Loki + Tempo + Grafana |
|------|------------------------|-------------------------------------|
| **통합성** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **표준화** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **성능** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **운영 복잡도** | ⭐⭐⭐⭐ | ⭐⭐ |
| **선택** | ✅ | ❌ |

**선택 이유**: OpenTelemetry는 업계 표준이며, SigNoz는 로그/메트릭/트레이스를 하나의 플랫폼에서 통합 관리할 수 있습니다.

#### MLOps 플랫폼

| 항목 | MLflow | Kubeflow | Weights & Biases |
|------|--------|----------|------------------|
| **실험 추적** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **모델 레지스트리** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **파이프라인** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **오픈소스** | ✅ | ✅ | ❌ |
| **선택** | ✅ (추적) | ✅ (파이프라인) | ❌ |

**선택 이유**: MLflow는 실험 추적과 모델 레지스트리에 우수하며, Kubeflow는 Kubernetes 네이티브 파이프라인 오케스트레이션에 적합합니다.

---

## 예상 이슈 및 해결 방안

### 1. GPU 리소스 부족

**문제**:
- 여러 ML 엔지니어가 동시에 GPU를 요청할 때 리소스 부족 발생
- 학습 작업과 개발 환경이 GPU를 경쟁적으로 사용

**해결 방안**:
- **다양한 GPU 할당 방식 활용**: 
  - MIG (A100/H100): 하드웨어 레벨 분할로 하나의 GPU를 여러 인스턴스로 분할
  - Time-Slicing: 시간 분할 방식으로 여러 Pod이 GPU 공유
  - 전체 GPU 할당: 성능이 중요한 작업에 전체 GPU 할당
  - 요구사항에 따라 적절한 방식 선택
- **리소스 큐 시스템**: GPU 요청을 큐에 넣고 우선순위 기반으로 할당
- **스케줄링 정책**: 
  - 개발 환경: 시간 제한 (예: 8시간 후 자동 종료)
  - 학습 작업: 우선순위 큐 (프로덕션 재학습 > 실험)
- **리소스 풀 분리**: 개발용 GPU 풀과 학습용 GPU 풀 분리

### 2. 모델 버전 관리 복잡성

**문제**:
- 모델 아티팩트가 크고 버전이 많아지면 저장소 용량 부족
- 모델과 코드 버전 간 불일치 발생

**해결 방안**:
- **모델 아티팩트 정책**: 
  - 오래된 모델 자동 아카이빙 (MinIO Lifecycle Policy)
  - 실험 단계 모델은 일정 기간 후 자동 삭제
- **버전 태깅**: Git 태그와 모델 버전을 연결하여 추적
- **모델 메타데이터**: 모델 레지스트리에 코드 커밋 해시 저장

### 3. 분산 학습 통신 오버헤드

**문제**:
- 여러 GPU 간 통신으로 인한 병목 현상
- 네트워크 대역폭 부족

**해결 방안**:
- **고속 네트워크**: InfiniBand 또는 고속 이더넷 사용
- **통신 최적화**: 
  - Gradient 압축 (Gradient Compression)
  - AllReduce 최적화 알고리즘 사용
- **로컬성 고려**: 같은 노드의 GPU를 우선적으로 할당
- **비동기 업데이트**: 완전 동기화 대신 비동기 SGD 고려

### 4. 자동 재학습 트리거 최적화

**문제**:
- 데이터 임계값 설정이 어려움 (너무 낮으면 빈번한 재학습, 너무 높으면 반영 지연)
- 재학습 중 리소스 경쟁

**해결 방안**:
- **적응형 임계값**: 
  - 데이터 품질과 양을 모두 고려
  - 모델 성능 저하 감지 시 임계값 낮춤
- **스케줄링**: 재학습을 비피크 시간에 실행
- **점진적 학습**: 전체 재학습 대신 증분 학습 고려

### 5. Canary 배포 실패 시 롤백

**문제**:
- 새 모델 배포 후 성능 저하 감지 시 빠른 롤백 필요
- 롤백 과정에서 서비스 중단 가능성

**해결 방안**:
- **자동 롤백**: 
  - 실시간 메트릭 모니터링 (SigNoz를 통한 지연시간, 에러율 등)
  - 임계값 초과 시 자동 롤백 트리거
- **Istio 트래픽 분할**: 점진적 전환 (10% → 50% → 100%)
- **블루-그린 배포**: 롤백 시 즉시 전환 가능한 구조

### 6. 데이터 품질 문제

**문제**:
- 라벨링 오류로 인한 모델 성능 저하
- 데이터 분포 변화 (Data Drift)

**해결 방안**:
- **데이터 검증**: 
  - 라벨링 데이터 자동 검증 (통계적 이상치 탐지)
  - 데이터 스키마 검증
- **Data Drift 감지**: 
  - 프로덕션 데이터와 학습 데이터 분포 비교
  - 자동 알림 및 재학습 트리거
- **라벨링 품질 관리**: 라벨러 간 일관성 검사

### 7. 개발 환경 격리 및 보안

**문제**:
- 여러 엔지니어가 같은 클러스터를 사용할 때 보안 및 격리 우려
- 민감한 데이터 접근 제어

**해결 방안**:
- **Pod 레벨 격리**: 공통 네임스페이스에서 Pod 단위 격리
- **Istio AuthorizationPolicy**: 서비스 간 접근 제어
- **Cilium Network Policy**: 네트워크 레벨 격리
- **RBAC**: 역할 기반 접근 제어로 리소스 접근 제한
- **Vault**: Kubernetes Secrets 또는 Vault를 통한 민감 정보 관리
- **이미지 스캔**: 컨테이너 이미지 보안 취약점 스캔

### 8. 모델 서빙 지연시간

**문제**:
- Real-time 서빙에서 높은 지연시간
- 동시 요청 처리 능력 부족

**해결 방안**:
- **모델 최적화**: 
  - 모델 양자화 (Quantization)
  - 모델 압축 (Pruning)
  - TensorRT, ONNX Runtime 등 최적화 런타임 사용
- **캐싱**: 자주 사용되는 예측 결과 캐싱 (Redis)
- **자동 스케일링**: KServe의 자동 스케일링 활용
- **배치 처리**: 가능한 경우 여러 요청을 배치로 처리

### 9. 비용 관리

**문제**:
- GPU 리소스 사용으로 인한 높은 비용
- 사용하지 않는 리소스에 대한 비용 발생

**해결 방안**:
- **리소스 모니터링**: SigNoz를 통한 사용량 추적 및 비용 분석 대시보드
- **자동 스케일 다운**: 사용하지 않는 개발 환경 자동 종료 (TTL 기반)
- **유연한 GPU 할당**: 
  - MIG, Time-Slicing 등으로 GPU 활용률 향상
  - 작업 특성에 맞는 할당 방식 선택
- **리소스 할당량**: 엔지니어별 GPU 사용 시간 제한
- **비용 알림**: 예산 초과 시 자동 알림

### 10. 재현성 보장

**문제**:
- 동일한 코드와 데이터로도 다른 결과 발생
- 환경 차이로 인한 재현성 문제

**해결 방안**:
- **컨테이너 이미지 버전 관리**: 학습에 사용된 정확한 환경 저장
- **시드 고정**: 랜덤 시드 고정으로 재현성 보장
- **의존성 고정**: Python 패키지 버전 명시 (requirements.txt, poetry.lock)
- **하드웨어 추적**: 사용된 GPU 모델 및 드라이버 버전 기록
- **실험 메타데이터**: MLflow에 모든 실험 파라미터와 환경 정보 자동 기록

### 11. 멀티클러스터 관리 복잡성

**문제**:
- 여러 클러스터 간 일관성 유지 어려움
- 클러스터 간 통신 및 보안 관리

**해결 방안**:
- **Fleet + ArgoCD**: 중앙화된 클러스터 및 애플리케이션 관리
- **Istio Multi-Primary**: 클러스터 간 서비스 메시 통합
- **정책 자동화**: GitOps를 통한 정책 자동 배포
- **모니터링 통합**: SigNoz를 통한 멀티클러스터 관찰 가능성

### 12. 관찰 가능성 데이터 과부하

**문제**:
- 대량의 로그, 메트릭, 트레이스 데이터로 인한 저장소 부담
- 쿼리 성능 저하

**해결 방안**:
- **샘플링**: 트레이스 샘플링으로 데이터 양 감소
- **데이터 보존 정책**: 오래된 데이터 자동 아카이빙
- **ClickHouse 최적화**: SigNoz의 ClickHouse 기반 고성능 쿼리
- **인덱싱**: 자주 조회하는 데이터 인덱싱

---

## 구현 계획

### Phase 1: 기본 인프라 (1-2개월)

1. **네임스페이스 및 기본 리소스**
   - 네임스페이스 생성 (mlops-dev, mlops-training, mlops-serving 등)
   - ResourceQuota 및 LimitRange 설정
   - 기본 네트워크 정책

2. **CRD 정의**
   - DevEnvironment CRD 정의
   - Custom Resource 스키마 설계

3. **멀티클러스터 관리**
   - Fleet 설치 및 클러스터 등록
   - ArgoCD 설치 및 구성

### Phase 2: 서비스 메시 & 보안 (2-3개월)

1. **Istio 설치**
   - Istio Multi-Primary 구성
   - mTLS 활성화
   - Gateway 및 VirtualService 구성

2. **네트워크 보안**
   - Cilium CNI 설치
   - 네트워크 정책 정의
   - AuthorizationPolicy 구성

3. **Identity & Access**
   - Keycloak 설치 및 구성
   - Vault 설치 및 Kubernetes 통합
   - OIDC 연동

### Phase 3: 관찰 가능성 (1-2개월)

1. **OpenTelemetry**
   - OpenTelemetry Operator 설치
   - OpenTelemetry Collector 배포
   - Istio와 OpenTelemetry 통합

2. **SigNoz**
   - SigNoz 설치 및 구성
   - ClickHouse 스토리지 설정
   - 대시보드 구성

### Phase 4: 개발 환경 관리 (2-3개월)

1. **Controller 구현**
   - DevEnvironment Controller 개발
   - Port Manager 구현
   - Resource Scheduler 구현

2. **API 서버**
   - REST API 서버 구현
   - 환경 생성/조회/삭제 API
   - 인증 및 권한 관리

3. **GPU 할당 설정**
   - GPU 노드 구성 (MIG, Time-Slicing 등)
   - 다양한 GPU 할당 방식 지원
   - Device Plugin 구성

### Phase 5: MLOps 플랫폼 (2-3개월)

1. **MLflow**
   - MLflow 서버 설치
   - PostgreSQL 백엔드 구성
   - 모델 레지스트리 설정

2. **Kubeflow Pipelines**
   - Kubeflow 설치
   - 파이프라인 컴포넌트 개발
   - 자동 재학습 트리거 구현

3. **KServe**
   - KServe 설치
   - 모델 서빙 인프라 구축
   - Canary 배포 구현

### Phase 6: 통합 및 테스트 (1-2개월)

1. **통합 테스트**
   - 전체 워크플로우 테스트
   - 성능 테스트
   - 보안 테스트

2. **문서화**
   - 사용자 가이드 작성
   - 운영 매뉴얼 작성
   - 트러블슈팅 가이드

### Phase 7: 운영 및 모니터링 (지속)

1. **모니터링**
   - SigNoz 대시보드 최적화
   - 알림 규칙 설정
   - 성능 메트릭 수집

2. **최적화**
   - 리소스 사용량 최적화
   - 비용 최적화
   - 성능 튜닝

---

## 결론

본 MLOps 플랫폼은 다음과 같은 특징을 가집니다:

1. **확장성**: 멀티클러스터 지원 및 수백 명의 개발자 지원
2. **보안**: 제로 트러스트 네트워킹 및 세밀한 접근 제어
3. **효율성**: 다양한 GPU 할당 방식 지원 및 자동화된 리소스 관리
4. **관찰 가능성**: OpenTelemetry + SigNoz를 통한 통합 관찰 가능성
5. **자동화**: GitOps 기반 배포 및 자동 재학습
6. **안정성**: Canary 배포 및 자동 롤백

이 플랫폼을 통해 ML 엔지니어는 격리된 환경에서 효율적으로 모델을 개발하고, 자동화된 파이프라인을 통해 지속적으로 모델을 개선하며, 안전하게 배포할 수 있습니다.

