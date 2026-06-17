# 01. Overview

# Microsoft Fabric Private Networking & AutoScale

## 개요

본 프로젝트는 Microsoft Fabric 환경에서 Capacity 비용 최적화와 Zero-Trust 보안 요구사항을 동시에 만족하기 위한 AutoScale 아키텍처를 검토하고 구현한 내용을 정리한 문서입니다.

일반적인 클라우드 환경에서는 Fabric Capacity 사용량(CU)을 모니터링하고 필요 시 SKU를 자동으로 상향 또는 하향 조정하는 방식으로 비용을 최적화할 수 있습니다.

하지만 엔터프라이즈 환경에서는 다음과 같은 제약사항이 존재합니다.

* Public Endpoint 사용 제한
* Global URL 접근 제한
* Outbound All Deny 정책
* Azure IAM 통제
* Fabric Admin 권한 통제
* 서비스 간 최소 권한 원칙 적용

이러한 환경에서는 단순히 API를 호출하는 것만으로는 AutoScale을 구현할 수 없습니다.

Capacity 사용량 조회부터 SKU 변경까지 모든 통신 경로와 권한 체계를 설계해야 하며, Microsoft Fabric과 Azure의 여러 관리 영역을 동시에 이해해야 합니다.

---

# 프로젝트 목표

본 프로젝트의 목표는 다음과 같습니다.

### 1. Capacity 비용 최적화

Fabric Capacity는 SKU 크기에 따라 비용이 결정됩니다.

일정 시간 동안 사용량이 낮은 경우 Capacity를 축소하고,

부하가 증가하는 경우 Capacity를 확장하여 운영 비용을 최적화하는 것을 목표로 합니다.

---

### 2. Private Network 기반 운영

보안 정책상 Public Endpoint 사용이 제한된 환경에서도 Fabric 서비스를 사용할 수 있어야 합니다.

이를 위해 다음 항목을 Private Network 기반으로 구성합니다.

* Fabric Warehouse
* Fabric API
* Capacity Metrics
* SKU Resize API

---

### 3. 운영 자동화

운영자가 직접 Capacity 상태를 확인하고 SKU를 변경하는 작업을 자동화합니다.

자동화 대상은 다음과 같습니다.

* Capacity 사용량 확인
* 임계치 판단
* SKU Resize
* 상태 모니터링

---

# 핵심 문제

AutoScale 기능 자체는 생각보다 복잡하지 않습니다.

실제로 가장 어려웠던 부분은 AutoScale 코드가 아니라 네트워크와 권한 구조였습니다.

대표적으로 아래 문제들을 해결해야 했습니다.

---

## Fabric Warehouse 접근

AKS에서 Fabric Warehouse로 접근하기 위해서는 Workspace Level Private Link가 필요합니다.

구성 항목

* Workspace Private Link
* Private Endpoint
* Private DNS
* NSG
* Firewall

---

## Capacity Metrics 접근

Capacity 사용량(CU)을 조회하기 위해서는 Capacity Metrics 데이터셋에 접근해야 합니다.

구성 항목

* Capacity Metrics Workspace
* Semantic Model
* ExecuteQueries API
* Workspace 권한

---

## api.powerbi.com 접근

Capacity Metrics 조회를 위해서는 api.powerbi.com 호출이 필요합니다.

하지만 보안 정책상 Global Endpoint 사용이 제한될 수 있습니다.

이를 해결하기 위해 Tenant Level Private Link 구성을 검토해야 합니다.

구성 항목

* Fabric Tenant Settings
* Private Link Service
* Private Endpoint
* Private DNS
* Firewall

---

## Fabric Capacity 제어

SKU 변경을 위해서는 Azure Resource Manager API 호출이 필요합니다.

구성 항목

* Azure IAM
* Service Principal
* Managed Identity
* Contributor 권한

---

# 아키텍처 관점

본 프로젝트는 크게 두 개의 영역으로 나뉩니다.

---

## Data Plane

실제 데이터가 이동하는 영역

```text
AKS
 ↓
Private Endpoint
 ↓
Fabric Workspace
 ↓
Warehouse
```

주요 목적

* 데이터 조회
* Warehouse 접근
* Lakehouse 접근

---

## Control Plane

Capacity 관리와 AutoScale을 수행하는 영역

```text
AKS
 ↓
Capacity Metrics
 ↓
Power BI API
 ↓
Fabric API
 ↓
Azure ARM API
```

주요 목적

* Capacity 모니터링
* SKU 변경
* AutoScale

---

# 구현 방식

본 프로젝트에서는 다음 두 가지 방식을 검토하였습니다.

## 방식 1. AKS 기반 AutoScale

```text
AKS
 ↓
Capacity Metrics 조회
 ↓
부하율 계산
 ↓
SKU Resize API 호출
```

특징

* 구현 자유도가 높음
* 실시간 처리 가능
* 별도 네트워크 구성 필요

---

## 방식 2. Fabric Native AutoScale

```text
Capacity Metrics
 ↓
Eventstream
 ↓
Activator
 ↓
Notebook
 ↓
SKU Resize
```

특징

* Fabric 내부에서 처리
* Azure 리소스 의존성 감소
* SaaS 구조 특성상 제약 존재

---

# Lessons Learned

Microsoft Fabric AutoScale 구현에서 가장 어려운 부분은 코드 작성이 아니었습니다.

실제로는 다음과 같은 영역을 동시에 이해해야 했습니다.

* Microsoft Fabric
* Power BI API
* Azure IAM
* Private Link
* Private Endpoint
* Private DNS
* Capacity Metrics
* AKS 인증 구조
* Fabric Admin 설정

특히 Workspace Level Private Link와 Tenant Level Private Link의 차이를 이해하지 못하면 설계 자체가 불가능합니다.

본 저장소는 이러한 시행착오와 검증 결과를 정리하여 향후 유사한 환경에서 참고할 수 있도록 작성되었습니다.

---

# 다음 문서

* 02-fabric-private-link.md
* 03-capacity-metrics.md
* 04-autoscale-architecture.md
* 05-tenant-vs-workspace.md
* 06-troubleshooting.md
