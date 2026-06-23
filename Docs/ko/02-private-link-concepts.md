# 02. Private Link Concepts

# 개요

Microsoft Fabric Private Networking을 이해하기 위해서는 먼저 Azure Private Networking의 핵심 구성 요소를 이해해야 합니다.

Microsoft Fabric에서 Workspace-Level Private Link 또는 Tenant-Level Private Link를 구성할 때 실제로 필요한 요소는 단순히 Private Link 하나가 아닙니다.

실제 구현 시에는 다음 요소들이 함께 동작해야 합니다.

* Private Link
* Private Endpoint
* Private DNS
* Private Link Service
* Virtual Network
* DNS Resolution
* Routing
* NSG / Firewall

이 문서에서는 Microsoft Fabric Private Networking을 구성하기 위해 필요한 핵심 개념들을 설명합니다.

---

# Private Networking이 필요한 이유

Enterprise 환경에서는 보안 정책상 Public Internet을 통한 SaaS 접근이 제한되는 경우가 많습니다.

대표적인 정책은 다음과 같습니다.

* Public Endpoint 차단
* Outbound All Deny
* 특정 Domain만 허용
* Private Connectivity 요구

이러한 환경에서는 Microsoft Fabric에 접근하기 위해 Private Networking 기반 연결 구성이 필요합니다.

예시:

```text id="0s7fqq"
AKS
 ↓
Private Network
 ↓
Microsoft Fabric
```

---

# Azure Private Link란?

Azure Private Link는 Azure 서비스 또는 SaaS 서비스에 Public Internet을 거치지 않고 Private IP 기반으로 접근할 수 있도록 지원하는 기능입니다.

즉,

Public Endpoint 대신 Private Endpoint를 통해 Azure Backbone Network 내부에서 통신하게 됩니다.

---

## Public Access

일반적인 Public Access 방식:

```text id="b6vwfu"
Client
 ↓
Internet
 ↓
Public Endpoint
 ↓
Service
```

특징:

* Public DNS 사용
* Internet 경유
* Firewall 정책 필요

---

## Private Access

Private Link 기반 Access 방식:

```text id="4aj34h"
Client
 ↓
Private IP
 ↓
Azure Backbone
 ↓
Service
```

특징:

* Private IP 사용
* Public Internet 우회
* Enterprise Security 강화

---

# Private Endpoint란?

Private Endpoint는 Virtual Network 내부에 생성되는 Private IP Endpoint입니다.

이 Endpoint를 통해 Azure 서비스 또는 SaaS 서비스와 Private 통신이 가능합니다.

예를 들어:

* Fabric Workspace
* Power BI API
* Storage
* SQL Database

등에 Private Endpoint를 연결할 수 있습니다.

---

## 예시

```text id="lgzt11"
AKS Pod
 ↓
10.x.x.x (Private Endpoint)
 ↓
Target Service
```

즉, Client는 Public IP가 아닌 Private IP를 통해 서비스에 접근하게 됩니다.

---

# Private DNS란?

Private Endpoint를 구성해도 DNS가 올바르게 설정되지 않으면 통신이 불가능합니다.

실제 운영 환경에서 가장 많이 발생하는 문제 중 하나가 DNS 문제입니다.

---

## DNS Resolution

예시:

```text id="8g27ep"
api.powerbi.com
   ↓
DNS Resolution
   ↓
10.x.x.x
```

즉,

도메인이 Private Endpoint의 Private IP로 Resolve되어야 합니다.

---

## DNS가 중요한 이유

Private Endpoint가 정상 생성되어도 DNS가 잘못 설정되면 다음 문제가 발생합니다.

* Public IP로 Resolve
* Connection Timeout
* Unexpected Routing

예시:

* Warehouse 접속 Timeout
* api.powerbi.com 접근 실패

---

# Private Link Service란?

Private Link Service (PLS)는 Private Link 연결의 Provider 역할을 수행합니다.

쉽게 말해:

* Service Provider 측 Endpoint
* Consumer가 Private Endpoint로 연결

구조입니다.

---

## 구조

```text id="z3owkg"
Provider Service
      ↓
Private Link Service
      ↓
Private Endpoint
      ↓
Client
```

Microsoft Fabric의 Tenant-Level Private Link에서는 이 개념이 중요하게 사용됩니다.

---

# Virtual Network

Private Endpoint는 반드시 Virtual Network 내부에 배치됩니다.

즉, Private Networking 설계 시 VNet 구조도 중요합니다.

예시:

* Hub-Spoke 구조
* Shared Services VNet
* AKS VNet

---

## 예시 구조

```text id="c8gnwy"
VNet
 ├── AKS Subnet
 ├── Private Endpoint Subnet
 └── Shared Services
```

---

# NSG (Network Security Group)

NSG는 네트워크 트래픽을 제어하는 Access Control Layer입니다.

예시 정책:

* Allow 1433
* Allow 443
* Deny All Others

---

## 주요 영향

NSG 정책이 잘못되면 다음 문제가 발생할 수 있습니다.

* SQL Port 차단
* HTTPS 차단
* Private Endpoint 통신 실패

---

# Firewall

Enterprise 환경에서는 Firewall 정책도 매우 중요합니다.

대표 정책:

* Outbound Allow List
* Domain Restriction
* Port Restriction

---

## 주요 영향

Firewall 정책이 잘못되면:

* api.powerbi.com 차단
* Fabric Endpoint 차단
* Connectivity Failure

---

# Routing

Private Networking 환경에서는 Routing도 중요합니다.

잘못된 Routing은 다음 문제를 유발합니다.

* Private Endpoint 우회
* Unexpected Public Access
* Connectivity Failure

---

# Microsoft Fabric에서 중요한 이유

Microsoft Fabric Private Networking을 구성할 때 위 모든 요소가 함께 작동해야 합니다.

예를 들어 Workspace-Level Private Link에서는:

* Private Endpoint
* Private DNS
* NSG
* Firewall

구성이 필요합니다.

Tenant-Level Private Link에서도 동일하게:

* Private Endpoint
* Private DNS
* Routing
* Firewall

구성이 필요합니다.

---

# 실제 장애의 대부분은 어디서 발생하는가?

실제 운영 환경에서는 대부분 아래 영역에서 장애가 발생합니다.

---

## 1. DNS

가장 흔한 문제

예시:

* Private Endpoint 생성 완료
* DNS 설정 누락
* 연결 실패

---

## 2. Network Policy

예시:

* NSG 차단
* Firewall 차단

---

## 3. Routing

예시:

* Public Route 사용
* Private Route 누락

---

# Key Takeaway

Microsoft Fabric Private Networking에서 가장 중요한 사실은 다음과 같습니다.

> Private Link 구성 자체보다 DNS, Routing, Security Policy 구성이 더 중요하다.

실제 대부분의 장애는 Private Link 생성 실패가 아니라 아래 문제에서 발생합니다.

* DNS Resolution 문제
* Routing 문제
* NSG / Firewall 정책 문제

다음 문서에서는 Microsoft Fabric Workspace-Level Private Link에 대해 자세히 다룹니다.

---

# 다음 문서

* 03-workspace-level-private-link.md
