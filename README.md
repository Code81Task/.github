# BDH — full project flowchart

This is one end-to-end view of the active Java application. Solid arrows show
the path a file takes; dashed arrows show shared control/configuration.
`[DB]` means the Oracle support tables accessed through `DBProcess` and
`BLHelper`. `alive.ctrl` is the process stop signal.

```mermaid
flowchart TB
    START([Start XMLRouter.main]) --> ALIVE{Can create<br/>ctrl/alive.ctrl?}
    ALIVE -- No --> STOP1([Stop: running or unclean<br/>instance detected])
    ALIVE -- Yes --> CONFIG[Load initialization.properties<br/>ApplicationParameters]
    CONFIG -. paths, flags, intervals .-> WORKERS
    CONFIG --> MODE{Startup mode}

    MODE -- normal --> RN[Start XML routing worker]
    MODE -- normal --> RL[Start loyalty worker]
    MODE -- simulation --> RS[Start XML routing worker<br/>simulation enabled]
    MODE -- simulation --> RSL[Start loyalty worker]
    MODE -- ignore --> RI[Start XML routing worker<br/>loyal/mail/load-balance checks off]
    MODE -- mail --> MP[Start mail-preparation worker]
    MODE -- mail --> MD[Start mail-delivery worker]
    MODE -- loadbalance --> LB[Start load-balance worker]
    MODE -- invalid --> STOP2([Log startup error])

    subgraph WORKERS[Long-running workers: each repeats while alive.ctrl exists]
      direction TB

      subgraph ROUTING[XML routing worker — normal / simulation / ignore]
        direction TB
        R0[Scan export TmpXML folder<br/>using configured finder command] --> R1{XML files found?}
        R1 -- No --> R2[Close DB connection<br/>sleep exportThreadSleep] --> R0
        R1 -- Yes --> R3[Open DB; process each XML]
        R3 --> R4[Read file name + size<br/>derive customer code]
        R4 --> R5{Simulation XML in a non-simulation run<br/>and customer not in CG samples?}
        R5 -- Yes --> R6[Remove XML] --> R3
        R5 -- No --> R7{Loyalty check enabled<br/>and customer is loyal?}
        R7 -- Yes --> R8[Write loyalty record<br/>to LOYALTY_FILES]
        R8 --> R9[Move XML to loyal temporary folder]
        R9 --> R3
        R7 -- No --> R10{Load balancing enabled<br/>and file exceeds maxFileSize?}
        R10 -- Yes --> R11[Classify XML type; write record with<br/>BDH_MODE = LOAD_BALANCE]
        R11 --> R12[Move XML to load-balance staging folder]
        R12 --> R3
        R10 -- No --> R13[Classify XML type<br/>NORMAL / EMAIL / EXCLUDE / CDBILL]
        R13 --> R14[Write/update BDH_FILES_WITH_MODE]
        R14 --> R15[Choose export input folder:<br/>in / email-in / exclude-in / email-exclude-in]
        R15 --> R16[Move XML to selected export input folder]
        R16 --> R3
      end

      subgraph LOYALTY[Loyalty worker — normal / simulation]
        direction TB
        L0[Query unprocessed calculated loyalty records<br/>for this export] --> L1{Records ready?}
        L1 -- No --> L2[Close DB connection<br/>sleep loyalThreadSleep] --> L0
        L1 -- Yes --> L3[For each loyalty XML:<br/>create points .txt in IncomingXML]
        L3 --> L4[Classify XML and choose input folder]
        L4 --> L5[Write/update BDH_FILES_WITH_MODE<br/>with loyal_flag = 1]
        L5 --> L6[Move XML from loyal temp<br/>to selected export input folder]
        L6 --> L7[Mark LOYALTY_FILES record processed] --> L0
      end

      subgraph BALANCING[Load-balance worker]
        direction TB
        B0[Query unprocessed records where<br/>BDH_MODE = LOAD_BALANCE] --> B1{Records ready?}
        B1 -- No --> B2[Close DB connection<br/>sleep exportThreadSleep] --> B0
        B1 -- Yes --> B3[Measure configured export folders]
        B3 --> B4[Select folder with lowest size]
        B4 --> B5[Move XML from staging to selected export;<br/>move loyalty points .txt when applicable]
        B5 --> B6[Choose selected export input folder]
        B6 --> B7[Mark record processed; save target destination] --> B0
      end

      subgraph MAILPREP[Mail-preparation worker]
        direction TB
        P0[Query email-type records not yet prepared] --> P1{Output PDF / CSV available?}
        P1 -- No or error --> P2[Record retry/failure;<br/>optional BDH mail log] --> P0
        P1 -- Yes --> P3[Create/update mail preparation record] --> P0
      end

      subgraph MAILDELIVERY[Mail-delivery worker]
        direction TB
        D0[Query prepared, unprocessed email records<br/>up to mailBatchSize] --> D1{Customer type}
        D1 -- Corporate --> D2[ZIP PDF + CSV; check ZIP size]
        D1 -- Consumer --> D3[Build consumer email<br/>from Velocity template + attachments]
        D2 --> D4[Resolve recipient and construct mail]
        D3 --> D4
        D4 --> D5{mailMode = archive?}
        D5 -- No --> D6[Send through SMTP]
        D5 -- Yes --> D7[Skip SMTP send]
        D6 --> D8{Send successful?}
        D8 -- No --> D9[Update retry/failure status;<br/>optional BDH mail log] --> D0
        D8 -- Yes --> D10[Update success/process status]
        D7 --> D10
        D10 --> D11[Archive generated ZIP, PDF and CSV] --> D0
      end
    end

    RN --> R0
    RS --> R0
    RI --> R0
    RL --> L0
    RSL --> L0
    LB --> B0
    MP --> P0
    MD --> D0

    R9 -. calculated by external billing process .-> L0
    R12 --> B0
    R16 -. export processing generates .-> OUTPUT[Export output<br/>PDF + CSV]
    B7 -. selected export processing generates .-> OUTPUT
    OUTPUT --> P0
    P3 --> D0

    DB[(Oracle support tables<br/>LOYALTY_FILES<br/>BDH_FILES_WITH_MODE<br/>BDH_MAIL_FILES_PREPARE<br/>BDH_MAIL_LOG)]
    R8 & R11 & R14 & L0 & L5 & L7 & B0 & B7 & P0 & P2 & P3 & D0 & D9 & D10 -. read / write .-> DB

    FILES[(Filesystem<br/>TmpXML · loyal temp · staging<br/>export input folders · IncomingXML<br/>mail temp · archive)]
    R0 & R6 & R9 & R12 & R16 & L3 & L6 & B5 & OUTPUT & D11 -. read / move / create .-> FILES

    CONTROL[ctrl/alive.ctrl] -. checked by every loop .-> ROUTING
    CONTROL -. checked by every loop .-> LOYALTY
    CONTROL -. checked by every loop .-> BALANCING
    CONTROL -. checked by every loop .-> MAILPREP
    CONTROL -. checked by every loop .-> MAILDELIVERY
```

How to read it:

1. Normal XML files go straight from `TmpXML` to the appropriate export input
   folder. Oversized files take the load-balancing path. Loyal files pause in
   the loyalty path until calculation is complete.
2. The export system (outside this source repository) consumes its input and
   produces PDF/CSV output. That output enters the mail-preparation and
   mail-delivery stages when the XML type is email-capable.
3. All workers use the same database records to coordinate progress. Removing
   `ctrl/alive.ctrl` cleanly ends their loops.

Primary code entry points: `XMLRouter` → `XMLThreadHandler` →
`XMLFileRouter`. `BLHelper` holds routing and SQL helper logic, `DBProcess`
executes database operations, `FileProcessor` runs filesystem commands, and
`LoadBalancer` selects the least-loaded export folder.
