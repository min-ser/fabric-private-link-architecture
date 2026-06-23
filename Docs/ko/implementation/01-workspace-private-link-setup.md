# 01. Workspace-Level Private Link Setup

# 개요

본 문서는 Microsoft Fabric Workspace-Level Private Link를 실제 엔터프라이즈 환경에서 구성하면서 확인한 내용을 정리한 구현 가이드입니다.

주요 목적은 다음과 같습니다.

* Fabric Warehouse를 Private IP 기반으로 접근
* Workspace-Level Private Link 구성 흐름 정리
* 공식 문서를 실제 고객사 환경에 적용할 때 필요한 아키텍처 이해 포인트 정리
* VNet, Subnet, Private Endpoint, Private DNS, Firewall, NSG 간 관계 설명

본 문서는 공식 문서를 단순히 요약한 문서가 아닙니다.

공식 문서에서 안내하는 절차를 실제 고객사 네트워크 환경에 적용하기 위해 어떤 구조를 먼저 이해해야 하는지를 중심으로 작성되었습니다.

---

# 공식 문서만으로는 어려웠던 이유

Microsoft 공식 문서에서는 Workspace-Level Private Link 구성을 위해 일반적으로 다음 흐름을 안내합니다.

1. Fabric Workspace 생성
2. Fabric Private Link Service 리소스 생성
3. Virtual Network 생성
4. Subnet 생성
5. Private Endpoint 생성
6. Private DNS 구성
7. VM 또는 Client에서 연결 확인
8. 필요 시 Public Access 차단

공식 문서는 기본적인 구성 절차를 이해하는 데는 도움이 됩니다.

하지만 실제 Enterprise 환경에서는 문서 그대로 적용하기 어려웠습니다.

그 이유는 고객사 환경이 공식 문서의 단순한 예제 환경과 다르기 때문입니다.

공식 문서의 예시는 일반적으로 다음과 같은 단순 구조를 가정합니다.

```text
Client VM
   ↓
VNet
   ↓
Subnet
   ↓
Private Endpoint
   ↓
Fabric Workspace
```

하지만 실제 고객사 환경은 보통 다음과 같이 훨씬 복잡합니다.

```text
AKS Pod
   ↓
AKS Subnet
   ↓
Spoke VNet
   ↓
Hub Firewall
   ↓
Private DNS / DNS Forwarder
   ↓
Private Endpoint Subnet
   ↓
Workspace-Level Private Link
   ↓
Fabric Workspace
   ↓
Fabric Warehouse
```

따라서 단순히 문서의 순서대로 리소스를 생성하는 것만으로는 연결이 보장되지 않습니다.

실제로는 다음 항목을 모두 이해해야 합니다.

* 기존 VNet 구조
* Subnet 분리 정책
* Private Endpoint 배치 위치
* Private DNS Resolution 흐름
* Hub-Spoke 구조
* Firewall 경유 여부
* NSG 정책
* Route Table
* AKS Pod에서의 DNS Resolution
* SQL 1433 통신 허용 여부

즉, Workspace-Level Private Link 구성의 핵심은 Portal에서 무엇을 클릭하는지가 아니라, 고객사 네트워크 아키텍처 안에서 해당 구성요소가 어떤 역할을 하는지 이해하는 것입니다.

---

# 이 문서에서 다루는 범위

본 문서는 Workspace-Level Private Link를 사용하여 Fabric Warehouse에 Private Network로 접근하는 시나리오를 다룹니다.

## 대상 시나리오

```text
AKS
 ↓
Private Endpoint
 ↓
Workspace-Level Private Link
 ↓
Fabric Workspace
 ↓
Fabric Warehouse
```

## 주요 목표

* AKS에서 Fabric Warehouse 접근
* Public Internet 경로 우회
* Private IP 기반 접근 확인
* Warehouse SQL Query 검증

## 주요 통신

| 구분           | 내용                          |
| ------------ | --------------------------- |
| Source       | AKS Pod, VM, On-Prem Client |
| Target       | Fabric Warehouse            |
| Network Path | Private Endpoint            |
| Protocol     | SQL                         |
| Port         | TCP 1433                    |
| Scope        | Workspace-Level             |

---

# Workspace-Level Private Link의 역할

Workspace-Level Private Link는 Fabric Workspace 수준에서 Data Endpoint에 대한 Private Access를 제공하는 구조입니다.

대표 대상은 다음과 같습니다.

* Fabric Warehouse
* Lakehouse
* SQL Analytics Endpoint

중요한 점은 Workspace-Level Private Link가 모든 Fabric 기능을 Private Network로 연결해주는 것은 아니라는 점입니다.

Workspace-Level Private Link의 목적은 Data Plane 접근입니다.

즉, 다음과 같은 작업에 사용됩니다.

* Warehouse SQL Query
* Lakehouse 접근
* SQL Analytics Endpoint 접근

반대로 다음 항목은 Workspace-Level Private Link만으로 해결되지 않습니다.

* api.powerbi.com 접근
* Power BI REST API 호출
* ExecuteQueries API 호출
* Capacity Metrics 조회
* Fabric Capacity Resize API 호출

이 항목들은 Tenant-Level Private Link 또는 별도 Outbound 허용 정책 검토가 필요합니다.

---

# 전체 구성 아키텍처

Workspace-Level Private Link 구성의 전체 흐름은 다음과 같습니다.

```text
Client / AKS Pod
      ↓
VNet Routing
      ↓
Private Endpoint IP
      ↓
Private DNS Resolution
      ↓
Workspace-Level Private Link
      ↓
Fabric Workspace
      ↓
Fabric Warehouse
```

실제 엔터프라이즈 환경에서는 다음과 같이 확장됩니다.

```text
AKS Pod
   ↓
AKS Node / Pod Network
   ↓
AKS Subnet
   ↓
Spoke VNet
   ↓
Hub Firewall / Route Table
   ↓
Private Endpoint Subnet
   ↓
Private DNS Zone
   ↓
Fabric Workspace Private Link
   ↓
Fabric Warehouse
```

---

# 구성 전 반드시 확인해야 할 항목

구성 전에 아래 항목을 먼저 확인해야 합니다.

## Fabric 측 확인

* Fabric Capacity 사용 여부
* Workspace가 Fabric Capacity에 할당되어 있는지
* Workspace ID 확인
* Tenant ID 확인
* Workspace Admin 권한 보유 여부
* Fabric Admin Tenant Setting 활성화 여부
* Workspace 인바운드 네트워크 설정 가능 여부

## Azure 측 확인

* Azure Subscription 권한
* Resource Group 생성 권한
* Private Endpoint 생성 권한
* Microsoft.Fabric Resource Provider 등록 여부
* VNet / Subnet 사용 가능 여부
* Private DNS Zone 생성 또는 연결 권한

## Network 측 확인

* Private Endpoint를 배치할 Subnet
* AKS 또는 Client가 위치한 VNet
* VNet Peering 여부
* Hub-Spoke 구조 여부
* Firewall 경유 여부
* NSG 정책
* Route Table
* DNS Forwarding 구조

## Client 측 확인

* AKS Pod에서 DNS 조회 가능 여부
* AKS Pod에서 TCP 1433 접근 가능 여부
* SQL Driver 설치 여부
* pyodbc 또는 pytds 사용 여부
* Service Principal 또는 사용자 인증 방식

---

# 구성 단계 요약

전체 구성 흐름은 다음과 같습니다.

```text
1. Fabric Workspace 준비
2. Workspace-Level Private Link 사용 조건 확인
3. Azure Fabric Private Link Service 리소스 생성
4. VNet / Subnet 준비
5. Private Endpoint 생성
6. Private DNS 구성
7. NSG / Firewall / Route Table 확인
8. AKS 또는 VM에서 DNS 검증
9. SQL 1433 연결 검증
10. Warehouse Query 검증
11. 필요 시 Public Access 차단
```

---

# Step 1. Fabric Workspace 준비

먼저 Fabric Workspace를 준비합니다.

확인 항목:

* Workspace 생성 여부
* Fabric Capacity 할당 여부
* Warehouse 또는 Lakehouse 생성 여부
* Workspace Admin 권한 여부

Workspace-Level Private Link는 Workspace 단위로 적용되므로, 어떤 Workspace를 Private Access 대상으로 구성할 것인지 명확해야 합니다.

또한 Warehouse 연결 테스트를 위해 대상 Warehouse가 미리 생성되어 있어야 합니다.

---

# Step 2. Workspace ID 및 Tenant ID 확인

Private Link Service 리소스를 생성할 때 Workspace ID와 Tenant ID가 필요합니다.

## Workspace ID

Workspace URL에서 확인할 수 있습니다.

예시:

```text
https://app.fabric.microsoft.com/groups/{workspace-id}/...
```

여기서 `{workspace-id}`가 Workspace ID입니다.

## Tenant ID

Tenant ID는 Fabric Portal 또는 Microsoft Entra ID에서 확인할 수 있습니다.

실제 구성 시 Tenant ID와 Workspace ID를 잘못 입력하면 Private Link Service 리소스가 생성되어도 정상 연결되지 않을 수 있습니다.

---

# Step 3. Microsoft.Fabric Resource Provider 확인

Workspace-Level Private Link를 처음 구성하는 Subscription에서는 Microsoft.Fabric Resource Provider 등록 상태를 확인해야 합니다.

확인 위치:

```text
Azure Portal
 → Subscription
 → Resource Providers
 → Microsoft.Fabric
```

필요 시 Re-register를 수행합니다.

이 단계가 누락되면 ARM Template 배포 또는 Fabric Private Link Service 리소스 생성 과정에서 실패할 수 있습니다.

---

# Step 4. Fabric Private Link Service 리소스 생성

Workspace-Level Private Link 구성에서는 Azure에 `Microsoft.Fabric/privateLinkServicesForFabric` 리소스를 생성합니다.

이 리소스는 Fabric Workspace와 Azure Private Endpoint를 연결하기 위한 대상 리소스 역할을 합니다.

개념적으로는 다음 구조입니다.

```text
Fabric Workspace
      ↓
Fabric Private Link Service Resource
      ↓
Private Endpoint
      ↓
VNet Client
```

예시 ARM Template 구조:

```json
{
  "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {},
  "resources": [
    {
      "type": "Microsoft.Fabric/privateLinkServicesForFabric",
      "apiVersion": "2024-06-01",
      "name": "<resource-name>",
      "location": "global",
      "properties": {
        "tenantId": "<tenant-id>",
        "workspaceId": "<workspace-id>"
      }
    }
  ]
}
```

주의할 점:

* `tenantId`는 Entra Tenant ID
* `workspaceId`는 Fabric Workspace ID
* location은 `global`
* 리소스 생성 후 Azure Portal에서 숨겨진 리소스로 표시될 수 있음

---

# Step 5. VNet 및 Subnet 설계

공식 문서에서는 연결 확인을 위해 VNet을 생성하고 Subnet을 할당하는 절차를 안내합니다.

하지만 실제 고객사 환경에서는 신규 VNet을 단순 생성하는 방식보다 기존 네트워크 구조에 어떻게 편입할지가 더 중요합니다.

## 왜 VNet이 필요한가?

Private Endpoint는 Azure Virtual Network 내부에 생성됩니다.

즉, Fabric Workspace에 Private IP로 접근하려면 Private Endpoint가 배치될 VNet이 필요합니다.

## 왜 Subnet이 필요한가?

Private Endpoint는 특정 Subnet에 배치됩니다.

실무에서는 일반적으로 Private Endpoint 전용 Subnet을 분리하는 것이 좋습니다.

예시:

```text
VNet
 ├── aks-subnet
 ├── private-endpoint-subnet
 └── shared-service-subnet
```

## Subnet 분리 이유

* Private Endpoint 관리 용이
* NSG 정책 분리
* Routing 정책 분리
* IP 관리 용이
* 운영 장애 영향 범위 축소

## IP 주소 고려사항

Workspace-Level Private Link는 여러 IP를 사용할 수 있으므로, Subnet 주소 범위를 너무 작게 잡으면 안 됩니다.

운영 환경에서는 향후 Workspace 추가, Endpoint 추가, 테스트 환경 분리까지 고려해 충분한 IP를 확보하는 것이 좋습니다.

---

# Step 6. Private Endpoint 생성

Private Endpoint는 Fabric Private Link Service 리소스를 대상으로 생성합니다.

Private Endpoint 생성 시 주요 설정은 다음과 같습니다.

| 항목                  | 설명                                            |
| ------------------- | --------------------------------------------- |
| Resource Type       | Microsoft.Fabric/privateLinkServicesForFabric |
| Target Sub-resource | workspace                                     |
| VNet                | Client 또는 AKS와 통신 가능한 VNet                    |
| Subnet              | Private Endpoint Subnet                       |
| DNS Integration     | Private DNS Zone 연동 권장                        |

Private Endpoint가 생성되면 해당 Subnet에 Private IP가 할당됩니다.

예시:

```text
10.x.x.x
```

이 IP가 이후 Fabric Workspace Endpoint에 대한 Private Access 경로가 됩니다.

---

# Step 7. Private DNS 구성

Workspace-Level Private Link 구성에서 가장 중요한 단계입니다.

Private Endpoint가 정상적으로 생성되어도 DNS가 올바르게 구성되지 않으면 연결은 실패합니다.

## 핵심 원칙

Fabric Workspace 관련 FQDN이 Public IP가 아니라 Private Endpoint IP로 Resolve되어야 합니다.

예시:

```text
{workspace-id}.z{xy}.w.api.fabric.microsoft.com
        ↓
Private DNS
        ↓
10.x.x.x
```

## Private DNS Zone

Workspace-Level Private Link에서는 일반적으로 다음 Private DNS Zone 구성이 필요합니다.

```text
privatelink.fabric.microsoft.com
```

Private DNS Zone은 Private Endpoint가 위치한 VNet뿐 아니라, 실제 Client가 위치한 VNet에서도 조회 가능해야 합니다.

## AKS 환경에서 특히 중요한 점

AKS Pod에서 DNS Resolution이 실제로 어떻게 동작하는지 반드시 확인해야 합니다.

검증은 Azure Portal에서 끝나지 않습니다.

반드시 AKS Pod 내부에서 확인해야 합니다.

예시:

```bash
nslookup <fabric-workspace-fqdn>
```

또는

```bash
dig <fabric-workspace-fqdn>
```

정상 결과:

```text
Name: <fabric-workspace-fqdn>
Address: 10.x.x.x
```

비정상 결과:

```text
Public IP로 Resolve
NXDOMAIN
Timeout
```

---

# Step 8. NSG / Firewall / Route Table 확인

Private Endpoint와 DNS가 구성되어도 네트워크 보안 정책에서 차단되면 연결은 실패합니다.

## NSG 확인

검토 대상:

* AKS Subnet Outbound
* Private Endpoint Subnet Inbound/Outbound
* TCP 1433 허용 여부
* TCP 443 허용 여부

Warehouse SQL 연결은 TCP 1433이 중요합니다.

## Firewall 확인

Enterprise 환경에서는 대부분 Firewall을 경유합니다.

확인 항목:

* AKS에서 Private Endpoint IP로 접근 가능한지
* TCP 1433이 허용되어 있는지
* DNS Query가 정상 처리되는지
* Forced Tunneling으로 인해 비정상 경로를 타지 않는지

## Route Table 확인

확인 항목:

* AKS Subnet Route Table
* Private Endpoint Subnet Route Table
* Hub-Spoke Peering
* UDR
* Firewall 경유 정책

Private Endpoint가 있어도 Route Table이 잘못되면 통신이 실패할 수 있습니다.

---

# Step 9. Connectivity Validation

연결 검증은 단계별로 수행해야 합니다.

바로 SQL Query를 실행하기보다 아래 순서로 점검하는 것이 좋습니다.

## 1단계. DNS 확인

```bash
nslookup <fabric-workspace-fqdn>
```

확인:

* Private IP로 Resolve되는지
* Public IP로 Resolve되지 않는지

## 2단계. Port 확인

```bash
nc -vz <fabric-workspace-fqdn> 1433
```

또는

```bash
telnet <fabric-workspace-fqdn> 1433
```

확인:

* TCP 1433 연결 가능 여부
* Timeout 여부
* Connection refused 여부

## 3단계. SQL Driver 확인

AKS Pod 또는 VM에 SQL Driver가 설치되어 있어야 합니다.

예시:

* ODBC Driver 18
* pyodbc
* pytds

## 4단계. Warehouse Query 확인

예시 코드:

```python
import pyodbc

conn_str = (
    "Driver={ODBC Driver 18 for SQL Server};"
    "Server=<fabric-warehouse-endpoint>,1433;"
    "Database=<database-name>;"
    "Authentication=ActiveDirectoryMsi;"
    "Encrypt=yes;"
    "TrustServerCertificate=no;"
)

conn = pyodbc.connect(conn_str)
cursor = conn.cursor()
cursor.execute("SELECT TOP 10 * FROM <schema>.<table>")

for row in cursor.fetchall():
    print(row)
```

환경에 따라 인증 방식은 달라질 수 있습니다.

예시:

* Service Principal
* Managed Identity
* Workload Identity
* User Authentication

---

# Step 10. Public Access 차단 검토

Private Link 연결이 정상적으로 검증된 후에는 필요에 따라 Workspace Public Access 차단을 검토할 수 있습니다.

주의할 점은 Public Access를 먼저 차단하면 문제 분석이 어려워질 수 있다는 점입니다.

권장 순서:

```text
1. Private Endpoint 구성
2. Private DNS 구성
3. AKS 또는 VM에서 연결 검증
4. Warehouse Query 성공 확인
5. Public Access 차단
6. 재검증
```

Public Access 차단 이후에는 Private Link 경로가 정상 동작하지 않으면 Workspace 접근 자체가 실패할 수 있습니다.

따라서 운영 환경에서는 반드시 검증 후 단계적으로 적용해야 합니다.

---

# 실제 구성 시 자주 발생한 문제

## 1. Private Endpoint는 Connected인데 통신 실패

증상:

* Azure Portal에서 Private Endpoint 상태는 Connected
* 하지만 AKS에서 Warehouse 접속 실패

주요 원인:

* DNS가 Private IP로 Resolve되지 않음
* NSG 차단
* Firewall 차단
* Route Table 문제

핵심:

```text
Private Endpoint Connected
≠
Application Connectivity Success
```

---

## 2. DNS가 Public IP로 Resolve됨

증상:

```text
nslookup 결과가 Private IP가 아님
```

주요 원인:

* Private DNS Zone 미연결
* VNet Link 누락
* DNS Forwarder 설정 누락
* AKS DNS 경로 미반영

해결 방향:

* Private DNS Zone 확인
* VNet Link 확인
* AKS Pod 내부에서 재확인
* 필요 시 CoreDNS / Custom DNS 확인

---

## 3. SQL 1433 Timeout

증상:

```text
Connection timeout
```

주요 원인:

* TCP 1433 차단
* Firewall 정책 차단
* UDR로 인한 우회 경로
* Private Endpoint Subnet 정책 문제

해결 방향:

* `nc -vz <endpoint> 1433`
* NSG Flow Log 확인
* Firewall Log 확인
* Route Table 확인

---

## 4. 공식 문서 기준 테스트 VM은 되는데 AKS는 실패

공식 문서에서는 VM을 생성하여 연결 확인하는 흐름을 안내할 수 있습니다.

하지만 실제 운영 환경에서는 AKS에서 접근해야 하는 경우가 많습니다.

VM에서는 되는데 AKS에서 안 되는 경우 다음을 확인해야 합니다.

* VM과 AKS가 같은 VNet인지
* AKS가 다른 Subnet에 있는지
* AKS Subnet NSG가 다른지
* AKS DNS가 Custom DNS를 사용하는지
* AKS Outbound가 Firewall을 경유하는지
* Pod Network에서 Private Endpoint까지 Route가 있는지

즉, VM 테스트 성공이 AKS 테스트 성공을 의미하지 않습니다.

---

# 운영 환경 적용 시 권장 접근 방식

공식 문서 절차를 그대로 수행하기보다, 다음 순서로 접근하는 것이 좋습니다.

## 1. 아키텍처 먼저 그리기

먼저 실제 고객사 네트워크 구조를 그립니다.

포함해야 할 요소:

* AKS VNet
* Private Endpoint Subnet
* Hub VNet
* Firewall
* DNS Forwarder
* Private DNS Zone
* Route Table
* Fabric Workspace

## 2. 통신 경로 정의

예시:

```text
AKS Pod
 → AKS Subnet
 → Route Table
 → Firewall
 → Private Endpoint IP
 → Fabric Workspace
```

## 3. DNS Resolution 경로 정의

예시:

```text
AKS Pod
 → CoreDNS
 → Custom DNS
 → Private DNS Zone
 → Private Endpoint IP
```

## 4. 보안 정책 정의

확인 항목:

* 1433 허용
* 443 허용
* DNS 허용
* Private Endpoint IP 허용
* 필요한 경우 FQDN 기반 정책 검토

## 5. 단계별 검증

검증 순서:

```text
DNS
 → Port
 → Authentication
 → SQL Connection
 → Query
```

---

# 검증 체크리스트

## Fabric

* [ ] Workspace가 Fabric Capacity에 할당되어 있음
* [ ] Workspace ID 확인
* [ ] Tenant ID 확인
* [ ] Workspace Admin 권한 있음
* [ ] Workspace-Level Private Link 관련 Tenant Setting 확인

## Azure

* [ ] Microsoft.Fabric Resource Provider 등록 확인
* [ ] Fabric Private Link Service 리소스 생성
* [ ] Private Endpoint 생성
* [ ] Target Sub-resource가 workspace인지 확인
* [ ] Private IP 할당 확인

## Network

* [ ] Private Endpoint Subnet 확인
* [ ] Private DNS Zone 생성 확인
* [ ] VNet Link 확인
* [ ] AKS VNet에서 DNS Resolve 가능
* [ ] NSG에서 1433 허용
* [ ] Firewall에서 1433 허용
* [ ] Route Table 확인

## AKS / Client

* [ ] Pod 내부에서 nslookup 성공
* [ ] Pod 내부에서 Private IP 확인
* [ ] TCP 1433 연결 성공
* [ ] SQL Driver 설치
* [ ] Warehouse 인증 성공
* [ ] Query 실행 성공

---

# Lessons Learned

Workspace-Level Private Link 구성에서 가장 어려웠던 부분은 Fabric 설정 자체가 아니었습니다.

실제 난이도는 고객사 네트워크에 있었습니다.

특히 다음 항목이 중요했습니다.

* VNet과 Subnet 구조
* Private Endpoint 위치
* Private DNS Resolution
* AKS DNS 경로
* NSG / Firewall 정책
* Route Table
* Hub-Spoke 구조

공식 문서는 기본 구성 절차를 안내하지만, 실제 고객사 환경은 훨씬 복잡합니다.

따라서 공식 문서를 적용하려면 먼저 전체 아키텍처를 이해해야 합니다.

핵심은 다음과 같습니다.

```text
공식 문서 = 기본 절차
실제 구축 = 고객사 네트워크 아키텍처에 맞춘 적용
```

---

# Key Takeaway

Workspace-Level Private Link는 Fabric Warehouse와 같은 Data Endpoint를 Private Network로 접근하기 위한 구조입니다.

하지만 Private Endpoint를 생성하는 것만으로는 충분하지 않습니다.

성공적인 구성을 위해서는 다음이 모두 맞아야 합니다.

* Fabric Workspace 설정
* Fabric Private Link Service 리소스
* Private Endpoint
* Private DNS
* VNet / Subnet
* NSG
* Firewall
* Routing
* AKS 또는 Client DNS Resolution
* SQL 1433 Connectivity

결론적으로 Workspace-Level Private Link 구축은 단순한 Fabric 설정 작업이 아니라, Enterprise Network Architecture 작업에 가깝습니다.
