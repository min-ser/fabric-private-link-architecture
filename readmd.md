```
├── README.md

├── docs

│   ├── 01-overview.md
│   ├── 02-fabric-private-link.md
│   ├── 03-capacity-metrics.md
│   ├── 04-autoscale-architecture.md
│   ├── 05-tenant-vs-workspace.md
│   ├── 06-troubleshooting.md

├── diagrams

│   ├── architecture.drawio
│   ├── private-link.drawio
│   ├── autoscale.drawio

├── sample

│   ├── dax-query.py
│   ├── fabric-resize.py
│   ├── metrics-check.py

└── images
```
# Microsoft Fabric Private Networking & AutoScale Reference Architecture

## 프로젝트 소개

본 저장소는 Microsoft Fabric 환경에서 Private Networking 기반 보안 아키텍처와 Capacity AutoScale 구현 과정에서 검증한 내용을 정리한 기술 문서입니다.

실제 엔터프라이즈 환경에서는 단순히 Fabric Warehouse 접근만 가능한 것이 아니라,

* Fabric Warehouse 접근
* Power BI API 접근
* Capacity Metrics 조회
* Fabric Capacity 제어
* AutoScale 구현

을 위해 다양한 네트워크, 권한, Fabric 관리 설정이 동시에 요구됩니다.

본 문서는 Microsoft Fabric Private Link 및 AutoScale 구현 과정에서 확인한 아키텍처와 고려사항을 정리하는 것을 목적으로 합니다.

---

# 검토 범위

본 저장소는 아래 항목을 중심으로 작성되었습니다.

### Fabric Secure Networking

* Workspace Level Private Link
* Tenant Level Private Link
* Private Link Service (PLS)
* Private Endpoint (PEP)
* Private DNS
* NSG
* Firewall
* AKS 연계

### Fabric Capacity Engineering

* Capacity Metrics
* DAX Query
* ExecuteQueries API
* Capacity Monitoring
* 429 Throttling 대응

### Fabric AutoScale

* AKS CronJob 방식
* Fabric Eventstream 방식
* Fabric Activator
* Fabric Notebook
* Fabric Capacity SKU Resize API

---

# 프로젝트 배경

기업 보안 환경에서는 일반적으로 Public Endpoint 사용이 제한됩니다.

Microsoft Fabric AutoScale을 구현하기 위해서는 Capacity 사용량(CU)을 수집해야 하며, 이를 위해 Power BI API(api.powerbi.com) 접근이 필요합니다.

하지만 보안 정책에 따라 글로벌 도메인 접근이 제한되는 경우가 많으며, 이러한 환경에서는 Private Link 기반의 별도 네트워크 구성이 필요합니다.

본 문서는 이러한 환경에서 Fabric AutoScale을 구현하기 위해 검토한 내용을 정리합니다.

---

# 핵심 아키텍처

## 1. Data Plane

Fabric Warehouse 접근

```text
AKS
 ↓
Private Endpoint
 ↓
Fabric Workspace
 ↓
Fabric Warehouse
```

주요 목적

* Warehouse 접근
* Lakehouse 접근
* 데이터 조회

주요 구성요소

* Workspace Level Private Link
* Private Endpoint
* Private DNS

---

## 2. Control Plane

Capacity Metrics 및 Fabric API 접근

```text
AKS
 ↓
api.powerbi.com
 ↓
Capacity Metrics
 ↓
ExecuteQueries
 ↓
Fabric API
```

주요 목적

* CU 사용량 조회
* Capacity 상태 확인
* SKU 변경

주요 구성요소

* Tenant Level Private Link
* Fabric Admin 설정
* Service Principal
* Azure IAM

---

# AutoScale 구현 방식

## 방식 1. AKS CronJob

```text
AKS CronJob
 ↓
Capacity Metrics 조회
 ↓
부하율 계산
 ↓
SKU Resize API 호출
```

장점

* 구현 자유도가 높음
* 로직 커스터마이징 가능
* 기존 플랫폼과 연계 용이

단점

* 네트워크 구성 필요
* 권한 설정 필요
* 운영 관리 필요

---

## 방식 2. Fabric Native Eventstream

```text
Capacity Metrics
 ↓
Eventstream
 ↓
Activator
 ↓
Notebook
 ↓
SKU 변경
```

장점

* Fabric Native 방식
* Azure 리소스 의존성 최소화

단점

* 실시간 반영 한계
* SaaS 제약 존재
* Azure Portal 제어와 분리

---

# 구현 시 필수 고려사항

Fabric AutoScale 구현 시 아래 항목이 모두 충족되어야 합니다.

## Fabric

* Fabric Admin 권한
* Tenant Setting 활성화
* Workspace 권한
* Capacity Metrics Workspace 권한

## Azure

* Azure IAM
* Contributor 권한
* Managed Identity
* Workload Identity

## Network

* Private Link Service
* Private Endpoint
* Private DNS
* NSG
* Firewall

## API

* ExecuteQueries API
* Capacity Metrics
* Fabric Capacity Resize API

---

# 주요 교훈 (Lessons Learned)

Microsoft Fabric AutoScale 구현의 난이도는 AutoScale 코드 자체에 있지 않았습니다.

실제로는 아래 항목을 모두 이해해야 구현이 가능했습니다.

* Workspace Level Private Link
* Tenant Level Private Link
* Fabric Admin 설정
* Capacity Metrics 권한
* AKS 인증 구조
* Azure IAM
* Private Networking
* Power BI API

특히 Fabric Warehouse 접근(Data Plane)과 Power BI API 접근(Control Plane)은 서로 다른 영역이며, 각 영역별로 요구되는 설정과 권한이 다릅니다.

따라서 AutoScale 구현 전 전체 아키텍처 흐름을 먼저 이해하는 것이 중요합니다.

---

# 참고 문서

Microsoft Fabric Workspace-Level Private Link

https://learn.microsoft.com/ko-kr/fabric/security/security-workspace-level-private-links-set-up

Microsoft Fabric Tenant + Workspace Private Link

https://learn.microsoft.com/ko-kr/fabric/security/security-tenant-workspace-private-links-same-tenant

Microsoft Fabric Real-Time Intelligence

https://learn.microsoft.com/ko-kr/fabric/real-time-intelligence

Microsoft Fabric Capacity Metrics

https://learn.microsoft.com/ko-kr/fabric/enterprise/metrics-app


---

# Microsoft Fabric Private Networking & AutoScale Reference Architecture

## Overview

This repository contains architectural considerations, implementation notes, and proof-of-concept findings for implementing Microsoft Fabric Capacity AutoScale in enterprise Zero-Trust environments.

The primary focus areas are:

* Microsoft Fabric Workspace-Level Private Link
* Microsoft Fabric Tenant-Level Private Link
* Fabric Capacity Metrics
* Power BI ExecuteQueries API
* Fabric Capacity SKU Resize
* AKS Integration
* Enterprise Network Security
* AutoScale Architecture

---

## Background

Many enterprise environments prohibit outbound access to public SaaS endpoints.

When implementing Fabric Capacity monitoring and AutoScale functionality, multiple networking, identity, and permission dependencies must be satisfied.

Examples:

* Fabric Admin Tenant Settings
* Workspace Permissions
* Power BI API Access
* Capacity Metrics Workspace Access
* Azure IAM Permissions
* Private Link Service
* Private Endpoint
* Private DNS
* NSG / Firewall Policies
* AKS Workload Identity

This repository documents architectural findings and implementation approaches discovered during real-world enterprise validation.

---

## Architecture Domains

### Data Plane

AKS → Fabric Warehouse

Used for:

* Data Access
* Warehouse Query
* Lakehouse Connectivity

Implemented using:

* Workspace-Level Private Link
* Private Endpoint
* Private DNS

---

### Control Plane

AKS → api.powerbi.com

Used for:

* Capacity Metrics Query
* ExecuteQueries API
* Fabric Management API

Implemented using:

* Tenant-Level Private Link
* Power BI API
* Fabric Admin Settings
* Azure IAM

---

## AutoScale Architecture

### Option A

AKS CronJob

AKS
↓
Capacity Metrics
↓
Threshold Evaluation
↓
Fabric Capacity Resize API

---

### Option B

Fabric Native Eventstream

Capacity Metrics
↓
Eventstream
↓
Activator
↓
Notebook
↓
SKU Resize

---

## Key Findings

### Workspace-Level Private Link is not always enough

Many implementations require understanding the difference between:

* Workspace-Level Private Link
* Tenant-Level Private Link

Depending on whether the target is:

* Warehouse Endpoint
* Power BI API
* Fabric Capacity Metrics
* Fabric Management APIs

---

### Private Networking involves multiple dependencies

Successful implementation requires:

* Networking
* Identity
* Fabric Administration
* Azure Administration
* Capacity Permissions

All components must be configured correctly before communication becomes possible.

---

## Disclaimer

This repository contains generalized architecture and implementation guidance.

No customer-specific information, resource identifiers, IP addresses, tenant IDs, or confidential operational details are included.
