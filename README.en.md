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

This repository documents architectural concepts, implementation notes, and real-world enterprise use cases for Microsoft Fabric Private Networking.

The primary focus is understanding the differences between:

* Workspace-Level Private Link
* Tenant-Level Private Link

Although both use Azure Private Link, they solve completely different connectivity requirements in Microsoft Fabric.

---

# Why This Repository Exists

Microsoft Fabric is a SaaS platform, but many enterprise environments operate under strict Zero-Trust security policies.

Common restrictions include:

* No Public Internet Access
* Outbound Deny-All Policy
* Restricted DNS Resolution
* Strict Firewall Policies
* Private Connectivity Requirements

Under these conditions, understanding Microsoft Fabric Private Networking becomes critical.

---

# Key Question

One of the most common misconceptions is:

> If Workspace-Level Private Link is configured, all Fabric communication should work.

This is incorrect.

Microsoft Fabric networking should be understood in two separate scopes.

---

## Workspace-Level Private Link

Used for private connectivity to data endpoints.

Supported workloads:

* Fabric Warehouse
* Lakehouse
* SQL Analytics Endpoint

Example:

```text
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

## Tenant-Level Private Link

Used for private connectivity to control-plane APIs.

Supported workloads:

* api.powerbi.com
* Power BI REST API
* ExecuteQueries API
* Capacity Metrics

Example:

```text
AKS
 ↓
Private Endpoint
 ↓
Tenant-Level Private Link
 ↓
api.powerbi.com
```

---

# Key Difference

| Target                 | Workspace-Level | Tenant-Level |
| ---------------------- | --------------- | ------------ |
| Fabric Warehouse       | O               | X            |
| Lakehouse              | O               | X            |
| SQL Analytics Endpoint | O               | X            |
| api.powerbi.com        | X               | O            |
| Capacity Metrics       | X               | O            |
| ExecuteQueries API     | X               | O            |

---

# Real-World Use Cases

This repository includes real-world enterprise validation scenarios.

### Use Case 1: Fabric Warehouse Private Access

AKS Pod → Private IP → Fabric Warehouse

* Private SQL connectivity
* Warehouse query execution
* Port 1433 communication

---

### Use Case 2: Power BI API Private Access

AKS Pod → Private IP → api.powerbi.com

* Capacity Metrics query
* ExecuteQueries API
* Power BI REST API

---

# Repository Structure

```bash
docs/
 ├── ko/
 └── en/

diagrams/
sample/
images/
```

---

# Documentation

## Korean Docs

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

# Key Takeaway

The most important architectural insight is:

> Workspace-Level Private Link and Tenant-Level Private Link solve completely different connectivity requirements.

Understanding this distinction is essential for implementing Microsoft Fabric in secure enterprise environments.

---

# Disclaimer

This repository contains generalized architectural guidance and implementation notes.

No customer-specific information, resource identifiers, IP addresses, tenant IDs, or confidential operational details are included.
