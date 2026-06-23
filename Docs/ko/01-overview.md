# 01. Overview

# 개요

본 문서는 Microsoft Fabric 환경에서 Private Networking 아키텍처를 설계하고 운영하면서 검증한 내용을 정리한 기술 문서입니다.

특히 다음 두 가지를 중심으로 다룹니다.

* Workspace-Level Private Link
* Tenant-Level Private Link

Microsoft Fabric는 SaaS 기반 서비스이지만, 엔터프라이즈 환경에서는 Public Endpoint 기반 접근이 제한되는 경우가 많습니다.

이러한 환경에서는 Fabric 서비스 접근을 위해 Private Link 기반 아키텍처가 필수적으로 요구됩니다.

본 문서는 실제 Enterprise 환경에서 Microsoft Fabric Private Networking을 검토하고 구현하며 확인한 아키텍처적 특징과 운영 시 고려사항을 정리하는 것을 목적으로 합니다.

---

# 프로젝트 배경

본 문서의 검토 배경은 다음과 같습니다.

Enterprise 환경에서 Microsoft Fabric를 운영하면서 아래 요구사항이 존재했습니다.

* AKS 환경에서 Fabric Warehouse 접근
* Private IP 기반 통신 요구
* Public Internet 차단
* Outbound All Deny 정책
* Power BI API 접근 제어
* Capacity 사용량 모니터링
* Capacity AutoScale 구현 검토

초기에는 단순히 Fabric Workspace에 Private Link를 구성하면 모든 요구사항을 충족할 수 있을 것으로 예상했습니다.

하지만 실제 검토 결과, Microsoft Fabric Networking 구조는 예상보다 훨씬 복잡했습니다.

---

# 왜 어려운가?

가장 어려웠던 점은 Microsoft Fabric의 모든 통신이 동일한 경로를 사용하지 않는다는 점입니다.

예를 들어 아래 두 작업은 완전히 다른 네트워크 경로를 사용합니다.

---

## Case 1. Fabric Warehouse 접근

AKS에서 Fabric Warehouse에 SQL Query를 수행하는 경우

```text id="90u42y"
AKS Pod
   ↓
Private Endpoint
   ↓
Fabric Workspace
   ↓
Warehouse
```

예시:

* pyodbc
* pytds
* SQL Port 1433

주요 목적:

* 데이터 조회
* Warehouse Query
* SQL Analytics Endpoint 접근

---

## Case 2. Capacity Metrics 조회

AKS에서 Capacity Metrics를 조회하는 경우

```text id="t27h9q"
AKS Pod
   ↓
api.powerbi.com
   ↓
ExecuteQueries API
   ↓
Capacity Metrics
```

예시:

* REST API
* ExecuteQueries API
* DAX Query

주요 목적:

* Capacity 사용량 조회
* Capacity 상태 모니터링
* AutoScale 판단

---

겉보기에는 둘 다 Microsoft Fabric 관련 작업이지만, 실제로는 완전히 다른 영역입니다.

---

# 핵심 아키텍처 개념

Microsoft Fabric Networking은 크게 두 개의 Scope로 나누어 이해해야 합니다.

* Workspace-Level Private Link
* Tenant-Level Private Link

이 둘은 목적이 완전히 다릅니다.

---

# Workspace-Level Private Link

Workspace-Level Private Link는 Fabric Workspace 수준의 Data Endpoint 접근을 위한 구조입니다.

대표적인 연결 대상:

* Fabric Warehouse
* Lakehouse
* SQL Analytics Endpoint

주요 목적:

* 데이터 접근
* Private SQL 통신
* Query 실행

---

# Tenant-Level Private Link

Tenant-Level Private Link는 Tenant 수준의 API Endpoint 접근을 위한 구조입니다.

대표적인 연결 대상:

* api.powerbi.com
* ExecuteQueries API
* Capacity Metrics
* Power BI REST API

주요 목적:

* Metrics 조회
* API 호출
* Capacity 관리

---

# 가장 흔한 오해

실제 검토 과정에서 가장 많이 발생했던 오해는 다음과 같습니다.

> Workspace-Level Private Link만 구성하면 모든 Fabric 기능이 동작할 것이다.

하지만 실제는 다릅니다.

---

## 가능한 것

Workspace-Level Private Link 구성 시 가능:

* Fabric Warehouse 접근
* SQL Query
* Lakehouse 접근

---

## 불가능한 것

Workspace-Level Private Link만으로는 불가능:

* Capacity Metrics 조회
* ExecuteQueries API 호출
* api.powerbi.com 접근

즉,

```text id="m7fyh8"
Workspace-Level Private Link
≠
Power BI API Access
```

이 점이 Microsoft Fabric Networking에서 가장 중요한 포인트입니다.

---

# 실제 운영 환경에서 고려해야 할 요소

Private Networking 기반 Microsoft Fabric 환경에서는 다음 영역을 모두 고려해야 합니다.

---

## Network

* Private Link
* Private Endpoint
* Private DNS
* NSG
* Firewall

---

## Fabric

* Workspace 설정
* Tenant Settings
* Capacity 설정
* Metrics Workspace

---

## Authentication

* Service Principal
* Managed Identity
* Workload Identity

---

## Authorization

* Azure IAM
* Workspace Permission
* Fabric Admin Permission

---

# 주요 장애 사례

실제 운영 과정에서 자주 발생했던 장애는 다음과 같습니다.

---

## Data Plane 장애

* Warehouse Port 1433 Timeout
* Private DNS Resolution 실패
* Private Endpoint 연결 후 통신 불가

예시:

* AKS Pod에서 Warehouse 접근 실패
* SQL Client Timeout

---

## Control Plane 장애

* ExecuteQueries API 403
* Capacity Metrics 조회 실패
* api.powerbi.com 접근 불가

예시:

* Permission 부족
* Tenant Setting 미설정
* API Authorization 실패

---

# 문서 구성

본 저장소는 다음 순서로 구성되어 있습니다.

---

## 02. Private Link Concepts

Azure Private Link 관련 핵심 개념 설명

* Private Link
* Private Endpoint
* Private DNS
* Private Link Service

---

## 03. Workspace-Level Private Link

Workspace-Level Private Link 아키텍처 상세 설명

* 연결 대상
* 구성 요소
* 제약 사항

---

## 04. Tenant-Level Private Link

Tenant-Level Private Link 아키텍처 상세 설명

* 연결 대상
* 구성 요소
* 제약 사항

---

## 05. Use Cases

실제 적용 사례

* Fabric Warehouse Private Access
* Power BI API Private Access

---

## 06. Troubleshooting

실제 장애 및 해결 사례 정리

* DNS
* SQL Timeout
* ExecuteQueries 403
* Metrics Access Issues

---

# Key Takeaway

Microsoft Fabric Private Networking에서 가장 중요한 포인트는 다음과 같습니다.

> Workspace-Level Private Link와 Tenant-Level Private Link는 서로 완전히 다른 연결 요구사항을 해결한다.

즉,

* Fabric Warehouse 접근 문제
* Power BI API 접근 문제

는 서로 다른 문제입니다.

이 차이를 이해하는 것이 Enterprise 환경에서 Microsoft Fabric를 성공적으로 운영하기 위한 핵심입니다.
