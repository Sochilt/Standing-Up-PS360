# Standing-Up-PS360
#A one-pager architecture plan and maintenance change plan; an Implementation plan + change ticket for standing up PS360
flowchart LR
  A[Start] --> B{Mermaid on GitHub?}
  B -->|Yes| C[🎉 Renders!]
  B -->|No| D[Check the code fence is ```mermaid]
  flowchart LR
  %% ZONES
  subgraph Z1[Client Network]
    R1[Radiologist Workstation<br/>PowerScribe Client + PowerMic]
    EPIC[Epic Hyperspace/Hyperdrive]
    VC[Visage Thin Client]
  end

  subgraph Z2[Clinical App VLAN]
    subgraph PS[PowerScribe 360 App Tier]
      PSAPP1[PS360-APP01<br/>(IIS + Nuance Services)]
      PSAPP2[PS360-APP02<br/>(IIS + Nuance Services)]
    end

    subgraph IF[Interface Layer (HL7)]
      HL7IN[[HL7 IN (ADT/ORM)<br/>LLP 2575]]
      HL7OUT[[HL7 OUT (ORU)<br/>LLP 2575]]
    end

    subgraph IMG[Imaging Fabric]
      LBR[Laurel Bridge Router<br/>AE=LBRTRouter_AE<br/>DICOM 11112 (TLS)]
      VIS[Visage 7 Servers<br/>AE=VISAGE7_AE<br/>DICOM 11112 / HTTPS 443]
      PACS[PACS/VNA<br/>AE=PACS_VNA_AE<br/>DICOM 104]
    end
  end

  subgraph Z3[Database VLAN]
    SQLP[(SQL PRIMARY – Comm4<br/>TCP 1433)]
    SQLS[(SQL SECONDARY – Comm4<br/>TCP 1433)]
  end

  %% CLIENT PATHS
  R1 -- HTTPS 443 --> PSAPP1
  R1 -- HTTPS 443 --> PSAPP2
  VC -- HTTPS 443 --> VIS
  EPIC -. Deep Link / IAN (HTTPS 443) .-> VIS

  %% PS360 <-> SQL
  PSAPP1 -- SQL 1433 --> SQLP
  PSAPP2 -- SQL 1433 --> SQLP
  SQLP <-. AlwaysOn Replication (async) .-> SQLS

  %% HL7 FLOWS
  EPIC <-. LLP 2575 .-> HL7IN
  EPIC <-. LLP 2575 .-> HL7OUT
  HL7IN --> PSAPP1
  HL7IN --> PSAPP2
  PSAPP1 --> HL7OUT
  PSAPP2 --> HL7OUT

  %% DICOM ROUTING
  LBR <--> VIS
  LBR <--> PACS

