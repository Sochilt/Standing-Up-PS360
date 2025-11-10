# Standing-Up-PS360
#A one-pager architecture plan and maintenance change plan; an Implementation plan + change ticket for standing up PS360
```mermaid

flowchart LR
  %% ========= ZONES =========
  subgraph Z1["Client Network"]
    R1["Radiologist Workstation\nPowerScribe Client + PowerMic"]
    EPIC["Epic Hyperspace / Hyperdrive"]
    VC["Visage Thin Client"]
  end

  subgraph Z2["Clinical App VLAN"]
    subgraph PSTIER["PowerScribe 360 App Tier"]
      PSAPP1["PS360-APP01 (IIS + Nuance Services)"]
      PSAPP2["PS360-APP02 (IIS + Nuance Services)"]
    end

    subgraph HIF["HL7 Interface Layer"]
      HL7IN["HL7 IN  ADT / ORM\nLLP 2575"]
      HL7OUT["HL7 OUT ORU\nLLP 2575"]
    end

    subgraph IMG["Imaging Fabric"]
      LBR["Laurel Bridge Router\nAE = LBRTRouter_AE\nDICOM 11112 TLS"]
      VIS["Visage 7 Servers\nAE = VISAGE7_AE\nDICOM 11112 / HTTPS 443"]
      PACS["PACS / VNA\nAE = PACS_VNA_AE\nDICOM 104"]
    end
  end

  subgraph Z3["Database VLAN"]
    SQLP["SQL PRIMARY  Comm4\nTCP 1433"]
    SQLS["SQL SECONDARY  Comm4\nTCP 1433"]
  end

  %% ========= FLOWS =========
  %% Client paths
  R1 -- "HTTPS 443" --> PSAPP1
  R1 -- "HTTPS 443" --> PSAPP2
  VC -- "HTTPS 443" --> VIS
  EPIC -. "Deep Link / IAN (HTTPS 443)" .-> VIS

  %% PS360 <-> SQL
  PSAPP1 -- "SQL 1433" --> SQLP
  PSAPP2 -- "SQL 1433" --> SQLP
  SQLP <-. "AlwaysOn replication (async)" .-> SQLS

  %% HL7 flows
  EPIC <-. "LLP 2575" .-> HL7IN
  EPIC <-. "LLP 2575" .-> HL7OUT
  HL7IN --> PSAPP1
  HL7IN --> PSAPP2
  PSAPP1 --> HL7OUT
  PSAPP2 --> HL7OUT

  %% DICOM routing
  LBR <--> VIS
  LBR <--> PACS

  %% ========= LEGEND =========
  subgraph LEG["Legend"]
    L_APP["App tier (green)"]
    L_HL7["HL7 (blue)"]
    L_DB["Database (red)"]
  end

  %% ========= CLASSES =========
  classDef app fill:#e6f4ea,stroke:#2f855a,stroke-width:1.5,color:#1a4731;
  classDef hl7 fill:#e6f0ff,stroke:#2b6cb0,stroke-width:1.5,color:#1e3a8a;
  classDef db  fill:#ffe8e6,stroke:#c53030,stroke-width:1.5,color:#7f1d1d;
  classDef viewer fill:#f3e8ff,stroke:#6b21a8,stroke-width:1.2,color:#3b0764;

  %% apply classes
  class PSAPP1,PSAPP2,PSTIER app;
  class HL7IN,HL7OUT,HIF hl7;
  class SQLP,SQLS,Z3 db;
  class VIS,LBR,PACS,VC viewer;
  class L_APP app;
  class L_HL7 hl7;
  class L_DB db;
```

## Overview
This diagram shows a reference deployment for **PowerScribe 360 (PS360)** integrated with **Visage 7**, Epic, an HL7 interface layer, and a DICOM router. It models redundant PS360 app servers, SQL AlwaysOn for the Comm4 database, HL7 ADT/ORM/ORU flows, DICOM routing to Visage/PACS, and Epic deep-link/Image Availability Notification (IAN).

## Scope & Assumptions
- Vendor apps are “vended systems”; IT configures, integrates, and operates (no source-code changes).
- PS360 app tier is two-node for resiliency; SQL AG is async (one primary, one secondary).
- Epic launches Visage with context; IAN is used to light up “images available” quickly.
- Router dual-sends PE-protocol CT to AI/Visage (optional) and all CT/MR to PACS/VNA.

## Component Glossary
- **PS360-APP01/02** – IIS + Nuance services (dictation, templates, reporting).
- **Comm4 (SQL AG)** – PowerScribe database (AlwaysOn).  
- **HL7 Interface Layer** – LLP 2575 channels: ADT/ORM in, ORU out to Epic.
- **Laurel Bridge Router** – DICOM/DICOMweb routing; may emit IAN.
- **Visage 7** – Server-side rendering; thin client launch from Epic.
- **PACS/VNA** – Long-term storage and enterprise distribution.

## Data Flow (high level)
1. **Order**: Epic → HL7 ORM; **ADT** for patient/visit.
2. **Acquisition**: Modality → Router → Visage/PACS; router optionally sends IAN to Epic.
3. **Read**: Radiologist views in Visage; dictates/finalizes in PS360.
4. **Report**: PS360 → HL7 ORU → Epic (and archive).  
5. **(Optional AI/SR)**: Router → AI service → DICOM SR/keys → Visage; mapped into PS360 templates.

## Ports, AEs, and Zones
| Layer | Purpose | Protocol/Port | Example IDs |
|---|---|---:|---|
| Client↔PS360 | App access | HTTPS **443** | — |
| HL7 IN (ADT/ORM) | Orders/demographics | LLP **2575** | Channel: `RAD_ADT_IN`, `RAD_ORM_IN` |
| HL7 OUT (ORU) | Final report to Epic | LLP **2575** | Channel: `RAD_ORU_OUT` |
| PS360↔SQL | Comm4 DB | SQL **1433** | Instance: `SQLCLUST01\PS360` |
| DICOM Router | Store/Move | DICOM **104/11112** | AE: `LBRTRouter_AE` |
| Visage | Viewer / DICOM | HTTPS **443** / **11112** | AE: `VISAGE7_AE` |
| PACS/VNA | Archive | DICOM **104** | AE: `PACS_VNA_AE` |

> **Zones:** Client Network (workstations), Clinical App VLAN (PS360/HL7/Router/Visage), Database VLAN (SQL AG). Restrict inter-zone traffic to required ports only; terminate TLS per policy.

## Change & Rollback (summary)
- **Pre-flight:** Full Comm4 backup, app snapshots, confirm client/server version compatibility.
- **Order of ops:** Stop PS360 services → DB upgrade (if required) → app upgrade APP01→APP02 → start services.
- **Smoke test:** Admin portal login; test dictation; sign report; verify ORU in Epic; verify Visage launch/latency.
- **Rollback:** Stop services → restore Comm4 backup → revert VM snapshot → keep clients on prior version.

## Validation Checklist
- ADT/ORM arrive and populate worklists; ORU ACKs clean (AA).
- Visage time-to-first-image within target (e.g., <2–3s on LAN).
- Router dual-send rules work (no loops); IAN fires within 30s.
- PS templates/macros load; microphones recognized (PowerMic/VDI).

## Notes for Reviewers
- This is a reference blueprint; replace AE titles, ports, and channel names with site-specific values.

```mermaid

flowchart LR
  %% ========= TITLE =========
  %% PS360 with Load Balancer/VIP, Health Checks, and Failover

  %% ========= ZONES =========
  subgraph Z1["Client Network"]
    R1["Radiologist Workstations / VDI"]
    DNS["DNS: ps360.myorg.local\n(A record -> VIP)"]
  end

  subgraph Z2["Clinical App VLAN"]
    LB["Load Balancer / VIP\nps360.myorg.local\nHTTPS 443"]
    subgraph PSTIER["PowerScribe 360 App Tier"]
      APP1["PS360-APP01\n(IIS + Nuance Services)"]
      APP2["PS360-APP02\n(IIS + Nuance Services)"]
    end

    subgraph HL7L["HL7 Interface Layer"]
      HL7IN["HL7 IN: ADT / ORM\nLLP 2575"]
      HL7OUT["HL7 OUT: ORU\nLLP 2575"]
    end
  end

  subgraph Z3["Database VLAN"]
    SQLP["SQL PRIMARY  (Comm4)\nTCP 1433"]
    SQLS["SQL SECONDARY (Comm4)\nTCP 1433"]
  end

  %% ========= CLIENT -> LB =========
  R1 --> DNS
  DNS --> LB
  R1 -- "HTTPS 443" --> LB

  %% ========= LB -> APP NODES =========
  LB -- "HTTPS 443" --> APP1
  LB -- "HTTPS 443" --> APP2

  %% ========= APP -> SQL =========
  APP1 -- "SQL 1433" --> SQLP
  APP2 -- "SQL 1433" --> SQLP
  SQLP <-. "AlwaysOn replication (async)" .-> SQLS

  %% ========= HL7 =========
  HL7IN --> APP1
  HL7IN --> APP2
  APP1 --> HL7OUT
  APP2 --> HL7OUT

  %% ========= HEALTH CHECKS (LB) =========
  subgraph HC["LB Health Checks (examples)"]
    HC1["HTTPS GET /radportal/login.aspx\nExpect 200 OK within < 2s"]
    HC2["TCP 443 open (IIS)"]
    HC3["Synthetic check: DB connect to Comm4\n(1433)"]
    HC4["Windows services running\n(Nuance* = Running)"]
  end

  HC1 -.-> APP1
  HC2 -.-> APP1
  HC3 -.-> APP1
  HC4 -.-> APP1

  HC1 -.-> APP2
  HC2 -.-> APP2
  HC3 -.-> APP2
  HC4 -.-> APP2

  %% ========= DRAIN / FAILOVER ANNOTATIONS =========
  LB -. "If a health check fails,\nLB marks node Unhealthy and stops new sessions" .- APP1
  LB -. "Use 'drain' to remove node for maintenance;\nexisting sessions finish, new ones go to healthy node" .- APP2

  %% ========= LEGEND =========
  subgraph LEG["Legend"]
    L_LB["Load balancer (orange)"]
    L_APP["PS360 app tier (green)"]
    L_HL7["HL7 (blue)"]
    L_DB["SQL Comm4 (red)"]
    L_HC["Dashed lines = health check probes"]
  end

  %% ========= CLASSES =========
  classDef lb  fill:#fff0da,stroke:#b45309,stroke-width:1.5,color:#7c2d12;
  classDef app fill:#e6f4ea,stroke:#2f855a,stroke-width:1.5,color:#1a4731;
  classDef hl7 fill:#e6f0ff,stroke:#2b6cb0,stroke-width:1.5,color:#1e3a8a;
  classDef db  fill:#ffe8e6,stroke:#c53030,stroke-width:1.5,color:#7f1d1d;
  classDef note fill:#f8fafc,stroke:#94a3b8,stroke-width:1.0,color:#334155;

  class LB,DNS,L_LB lb;
  class APP1,APP2,PSTIER,L_APP app;
  class HL7IN,HL7OUT,HL7L,L_HL7 hl7;
  class SQLP,SQLS,Z3,L_DB db;
  class HC,HC1,HC2,HC3,HC4,L_HC note;
```

