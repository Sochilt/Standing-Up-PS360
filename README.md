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

