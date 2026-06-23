```bash
fabric-private-link-architecture/
│
├── README.md
│
├── docs/
│   ├── ko/
│   │   ├── 01-overview.md
│   │   ├── 02-private-link-concepts.md
│   │   ├── 03-workspace-level-private-link.md
│   │   ├── 04-tenant-level-private-link.md
│   │   ├── 05-use-cases.md
│   │   └── 06-troubleshooting.md
│   │
│   └── en/
│       ├── 01-overview.md
│       ├── 02-private-link-concepts.md
│       ├── 03-workspace-level-private-link.md
│       ├── 04-tenant-level-private-link.md
│       ├── 05-use-cases.md
│       └── 06-troubleshooting.md
│
├── diagrams/
│   ├── overall-architecture.drawio
│   ├── workspace-level.drawio
│   └── tenant-level.drawio
│
├── sample/
│   ├── warehouse-query.py
│   ├── dax-query.py
│   └── metrics-check.py
│
└── images/
```
# Microsoft Fabric Private Link Architecture

## Workspace-Level & Tenant-Level Private Networking Reference

본 저장소는 Microsoft Fabric 환경에서 Private Networking 아키텍처를 설계하고 구현하기 위해 필요한 개념, 구성 요소, 실제 적용 사례를 정리한 기술 레퍼런스 문서입니다.

주요 목적은 Microsoft Fabric에서 사용되는 두 가지 Private Link 구조를 명확하게 구분하고 이해하는 것입니다.

* Workspace-Level Private Link
* Tenant-Level Private Link

두 방식 모두 Azure Private Link 기술을 기반으로 하지만,
연결 대상과 목적은 완전히 다릅니다.

---

# 왜 이 저장소가 필요한가

Microsoft Fabric는 SaaS 기반 서비스이지만,
대부분의 엔터프라이즈 환경에서는 엄격한 보안 정책이 적용됩니다.

대표적인 제약사항은 다음과 같습니다.

* Public Internet 접근 제한
* Outbound All Deny 정책
* 제한된 DNS Resolution
* 엄격한 Firewall 정책
* Private Connectivity 요구사항

이러한 환경에서는 Microsoft Fabric를 사용하기 위해 Private Networking 아키텍처를 정확하게 이해해야 합니다.

---

# 핵심 질문

가장 흔하게 발생하는 오해는 다음과 같습니다.

> Workspace-Level Private Link를 구성하면 Fabric의 모든 통신이 가능하다.

이는 사실이 아닙니다.

Microsoft Fabric의 Private Networking은 크게 두 개의 Scope로 나누어 이해해야 합니다.

---

# Workspace-Level Private Link

Fabric 데이터 엔드포인트에 대한 Private Connectivity를 제공합니다.

연결 가능한 주요 대상:

* Fabric Warehouse
* Lakehouse
* SQL Analytics Endpoint

예시:

```text id="qyl2f3"
AKS
 ↓
Private Endpoint
 ↓
Workspace-Level Private Link
 ↓
Fabric Workspace
 ↓
Warehouse
```

---

# Tenant-Level Private Link

Fabric 및 Power BI Control Plane API에 대한 Private Connectivity를 제공합니다.

연결 가능한 주요 대상:

* api.powerbi.com
* Power BI REST API
* ExecuteQueries API
* Capacity Metrics

예시:

```text id="pmv7go"
AKS
 ↓
Private Endpoint
 ↓
Tenant-Level Private Link
 ↓
api.powerbi.com
```

---

# 핵심 차이점

| 대상                     | Workspace-Level | Tenant-Level |
| ---------------------- | --------------- | ------------ |
| Fabric Warehouse       | O               | X            |
| Lakehouse              | O               | X            |
| SQL Analytics Endpoint | O               | X            |
| api.powerbi.com        | X               | O            |
| Capacity Metrics       | X               | O            |
| ExecuteQueries API     | X               | O            |

---

# 실제 적용 사례

본 저장소에는 실제 엔터프라이즈 환경에서 검증한 Private Networking 사례를 포함합니다.

---

## Use Case 1: Fabric Warehouse Private Access

AKS Pod → Private IP → Fabric Warehouse

주요 내용:

* Private SQL 통신
* Warehouse Query 실행
* Port 1433 기반 통신

---

## Use Case 2: Power BI API Private Access

AKS Pod → Private IP → api.powerbi.com

주요 내용:

* Capacity Metrics 조회
* ExecuteQueries API
* Power BI REST API

---

# Repository 구조

```bash id="2l2dm8"
docs/
 ├── ko/
 └── en/

diagrams/
sample/
images/
```

---

# 문서 구성

## 한국어 문서

* [01. Overview](docs/ko/01-overview.md)
* [02. Private Link Concepts](docs/ko/02-private-link-concepts.md)
* [03. Workspace-Level Private Link](docs/ko/03-workspace-level-private-link.md)
* [04. Tenant-Level Private Link](docs/ko/04-tenant-level-private-link.md)
* [05. Use Cases](docs/ko/05-use-cases.md)
* [06. Troubleshooting](docs/ko/06-troubleshooting.md)

---

## English Docs

* [01. Overview](docs/en/01-overview.md)
* [02. Private Link Concepts](docs/en/02-private-link-concepts.md)
* [03. Workspace-Level Private Link](docs/en/03-workspace-level-private-link.md)
* [04. Tenant-Level Private Link](docs/en/04-tenant-level-private-link.md)
* [05. Use Cases](docs/en/05-use-cases.md)
* [06. Troubleshooting](docs/en/06-troubleshooting.md)

---

# 핵심 메시지

이 저장소에서 가장 중요한 아키텍처 포인트는 다음과 같습니다.

> Workspace-Level Private Link와 Tenant-Level Private Link는 서로 완전히 다른 연결 요구사항을 해결합니다.

즉,

* Fabric Warehouse 접근
* Power BI API 접근

은 전혀 다른 문제입니다.

이 차이를 이해하는 것이 엔터프라이즈 환경에서 Microsoft Fabric를 안정적으로 운영하기 위한 핵심입니다.

---

# Disclaimer

본 저장소는 일반화된 아키텍처 가이드와 구현 경험을 정리한 문서입니다.

고객 환경의 Resource Identifier, IP, Tenant ID 및 기타 민감 정보는 포함하지 않습니다.
