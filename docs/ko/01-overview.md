# 01. Overview

# 개요

본 프로젝트는 Microsoft Fabric 환경에서 **Capacity 비용 최적화**와 **Enterprise Zero-Trust 보안 요구사항**을 동시에 만족하기 위한 AutoScale 아키텍처를 검토하고 구현한 내용을 정리한 기술 문서입니다.

일반적인 클라우드 환경에서는 Fabric Capacity 사용량(CU)을 모니터링하고, 필요 시 SKU를 자동으로 조정하여 비용을 최적화할 수 있습니다.

예를 들어:

* 사용량이 낮은 시간대 → Capacity 축소
* 사용량이 증가한 시간대 → Capacity 확장

이를 통해 불필요한 비용을 줄이면서도 안정적인 서비스 운영이 가능합니다.

하지만 엔터프라이즈 환경에서는 단순히 API 호출만으로 AutoScale을 구현하기 어렵습니다.

---

# 프로젝트 배경

대부분의 기업 보안 환경에서는 다음과 같은 제약사항이 존재합니다.

* Public Endpoint 사용 제한
* Global URL 접근 제한
* Outbound All Deny 정책
* Azure IAM 통제
* Fabric Admin 권한 통제
* 최소 권한(Least Privilege) 원칙 적용

이러한 환경에서는 Capacity 사용량 조회부터 SKU 변경까지 모든 통신 경로와 권한 구조를 사전에 설계해야 합니다.

즉, 단순히 AutoScale 코드만 작성해서는 구현이 불가능합니다.

Microsoft Fabric, Azure, Networking, Identity, Security 전반에 대한 이해가 필요합니다.

---

# 프로젝트 목표

본 프로젝트의 목표는 다음과 같습니다.

---

## 1. Capacity 비용 최적화

Fabric Capacity는 SKU 크기에 따라 비용이 결정됩니다.

Capacity 사용량이 낮을 경우 SKU를 축소하고,
부하가 증가하는 경우 SKU를 확장하여 비용 효율적인 운영을 목표로 합니다.

대표 예시:

* F32 → F16 (Scale Down)
* F16 → F32 (Scale Up)

---

## 2. Private Network 기반 운영

보안 정책상 Public Endpoint 사용이 제한된 환경에서도 Fabric 서비스를 안정적으로 사용할 수 있어야 합니다.

Private Networking 기반으로 검토 대상은 다음과 같습니다.

* Fabric Warehouse
* Fabric API
* Capacity Metrics
* SKU Resize API

---

## 3. 운영 자동화

운영자가 직접 Capacity 상태를 확인하고 SKU를 조정하는 작업을 자동화합니다.

자동화 대상:

* Capacity 사용량 확인
* 임계치 판단
* SKU Resize
* 상태 모니터링

---

# Architecture Domains

Microsoft Fabric AutoScale 아키텍처는 크게 두 개의 영역으로 나뉩니다.

* Data Plane
* Control Plane

이 두 영역은 목적과 네트워크 경로가 서로 다릅니다.

---

# Data Plane

Data Plane은 실제 데이터 접근을 담당하는 영역입니다.

대표 예시:

* Fabric Warehouse 접근
* Lakehouse 접근
* SQL Query 실행

아키텍처:

```text id="srwzb0"
AKS
 ↓
Private Endpoint
 ↓
Fabric Workspace
 ↓
Warehouse
```

주요 목적:

* 데이터 조회
* Warehouse 접근
* Lakehouse 접근

---

# Control Plane

Control Plane은 Capacity 관리 및 운영 기능을 담당하는 영역입니다.

대표 예시:

* Capacity Metrics 조회
* Power BI API 호출
* Fabric API 호출
* SKU Resize

아키텍처:

```text id="dj67vx"
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

주요 목적:

* Capacity 모니터링
* SKU 변경
* AutoScale 제어

---

# 구현 방식

본 프로젝트에서는 두 가지 AutoScale 구현 방식을 검토하였습니다.

---

## 방식 1. AKS 기반 AutoScale

AKS CronJob 또는 Scheduler 기반 방식입니다.

아키텍처:

```text id="l7d5n0"
AKS
 ↓
Capacity Metrics 조회
 ↓
부하율 계산
 ↓
SKU Resize API 호출
```

장점:

* 구현 자유도 높음
* 실시간 처리 가능
* 커스터마이징 용이

단점:

* 네트워크 구성 필요
* 권한 설정 필요
* 운영 관리 필요

---

## 방식 2. Fabric Native AutoScale

Fabric 내부 기능을 활용하는 방식입니다.

아키텍처:

```text id="m4k4f2"
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

장점:

* Fabric Native 방식
* Azure 리소스 의존성 감소

단점:

* SaaS 제약 존재
* 실시간 제어 한계
* Azure 제어와 분리

---

# 핵심 과제

AutoScale 구현에서 가장 어려운 부분은 AutoScale 로직이 아니었습니다.

실제 난이도는 아래 항목에 있었습니다.

---

## Fabric Warehouse 접근

AKS에서 Fabric Warehouse로 접근하기 위해 필요한 요소:

* Workspace-Level Private Link
* Private Endpoint
* Private DNS
* NSG
* Firewall

---

## Capacity Metrics 접근

Capacity 사용량(CU) 조회를 위해 필요한 요소:

* Capacity Metrics Workspace
* Semantic Model
* ExecuteQueries API
* Workspace Permission

---

## Power BI API 접근

Capacity Metrics 조회를 위해 필요한 요소:

* api.powerbi.com 접근
* Tenant-Level Private Link
* Tenant Setting
* Firewall 정책

---

## Fabric Capacity 제어

SKU 변경을 위해 필요한 요소:

* Azure ARM API
* Azure IAM
* Service Principal
* Managed Identity
* Contributor 권한

---

# Lessons Learned

Microsoft Fabric AutoScale 구현에서 가장 어려운 부분은 코드 작성이 아니었습니다.

실제로는 아래 영역을 동시에 이해해야 구현이 가능했습니다.

* Microsoft Fabric
* Power BI API
* Azure IAM
* Private Link
* Private Endpoint
* Private DNS
* Capacity Metrics
* AKS 인증 구조
* Fabric Admin 설정

특히 가장 중요한 점은 다음과 같습니다.

> Workspace-Level Private Link ≠ Power BI API Access

즉,

* Warehouse 접근 (Data Plane)
* Metrics/API 접근 (Control Plane)

은 서로 다른 문제입니다.

이 차이를 이해하지 못하면 AutoScale 아키텍처 설계가 매우 어려워집니다.

---

# Intended Audience

본 문서는 다음과 같은 엔지니어를 대상으로 작성되었습니다.

* Cloud Architect
* Azure Engineer
* Platform Engineer
* Fabric Engineer
* Security Engineer

특히 Enterprise Zero-Trust 환경에서 Microsoft Fabric를 운영하는 조직에 유용합니다.

---

# 다음 문서

* 02-fabric-private-link.md
