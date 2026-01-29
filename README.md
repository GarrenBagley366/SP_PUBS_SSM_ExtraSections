# SP_PUBS_SSM_ExtraSections

### The Extra Section Engine is a stand-alone SpringBoot/Kubernetes application running on a Production Publications Azure node. A SpringScheduler configuration prompts the Engine to regularly poll a SabreMQ via an IBM MQ schedule Upates. Extra sections are determined via flight range.  The Sabre green screen formatted data is effectively screen scraped and reformatted to an IATA standard SSM format in a file then transmitted to ITA via the existing NPPortal/EMFT file transfer system used for the Publications ITA SSIM transmits.

### Each extra session processed is archived to a dated file with the original green screen format and the resulting SSM format, a copy of this is included with a notification emailed to a configurable list of email recipients.


## Flowchart

```mermaid
flowchart TD
  %% Sources
  SABRE["SABRE Schedule change feed via IBM MQ"] --> MQ["IBM MQ"]

  %% Generator
  subgraph GEN["SP_PUBS_SSM_ExtraSections (Spring/Maven)"]
    MQ --> G1["Scheduled Trigger (cron/SpringScheduler) Consume IBM MQ messages"]
    G1 --> G2["Parse schedule message"]
    click G2 href "https://github.com/AAInternal/FltInvhub_Schedule_FileProcessor/wiki/File-Processor-Business-Rules"
    G2 --> G3["Insert/Update SCHEDULE_MESSAGE<br/>(status management)"]
    G2 -->|Original Format| G4["Store Flight in Original Screen Format for archive<br/>"]
    

G6["Validate prerequisites<br/>- not already generated today (SCHED_FILE_PROCESSING_COMPLETED_DATE)<br/>- FLIGHT_ID not running<br/>- MRU/INIT not in progress<br/>- weekend MRU completed / INIT completed timing checks"]
    G6 --> G7["Select PENDING messages<br/>sleep 30s<br/>re-select by clock-time window"]
    G7 --> G8["Build JSON schedule payload<br/>Schedule_File_* or Extra_Section_Schedule_File_*"]
    G8 --> G9["Mark messages PROCESSED"]
    G8 --> BLOB_IN["Azure Blob Container: fltinvhub-schedule"]
  end

  %% Message Bus
  subgraph ServiceBus
    SB["Optional File Move and Service Bus message<br>if breaking SP_PUBS_SSM_ExtraSections into two instances "]
  end

  subgraph DIR["/aa-pubs-extrasection"]
    FA["Extra_Section_Schedule_File_20260122_010031.json"]
  end

  %% Edges
  G3 --> SB
  G4 --> FA
  SB --> G6

```
