# SP_PUBS_SSM_ExtraSections

## The Extra Section Engine consumes IBM MQ messages and sends the extra sections to ITA google via EMFT

### The Extra Section Engine is a stand-alone SpringBoot/Kubernetes application running on a Production Publications Azure node. A SpringScheduler configuration prompts the Engine to regularly poll a SabreMQ via an IBM MQ schedule Upates. Extra sections are determined via flight range.  The Sabre green screen formatted data is effectively screen scraped and reformatted to an IATA standard SSM format in a file then each file is transmitted to ITA via EMFT with a new NPPortal/EMFT jobs ismilar to the existing NPPortal/EMFT file transfer system used for the Publications ITA SSIM transmits.

### Each extra session processed is archived to a dated file with the original green screen format and the resulting SSM format, a copy of this is included with a notification emailed to a configurable list of email recipients.

#### Assumptions
* Once moved to production there is no human interaction to decide if an extra section is published, this is unlike Publications SABRE SSM
* We may only do ADD and DELETE types
* We may only schedule Monday and Tuesday or just weekdays, no weekend processing to avoid MRU.
* The extra sections will not be enriched with Code share marketing data or enriched with Publications Logic.
* Will not have a UI, will communicate via Email to a list of stakeholders

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
G2 --> G2A["EBCDIC to ASCII encoding<br\> - <br\>Validate prerequisites<br/> - Matches Extra Section Flight Range
<pre>
Message starts with A → ssimType = ADD
Message starts with D → ssimType = DELETE
Message starts with I → ssimType = INIT
Message starts with 6 → ssimType = TYPE6
</pre><br/>
- not already generated today (SCHED_FILE_PROCESSING_COMPLETED_DATE)<br/>"]
    G2A --> G3["ADD/DEL SCHEDULE_MESSAGE<br/>(status management)"]
    G2A -->|Original Format| G4["Store Flight in Original Screen Format for archive<br/>"]
    
     G8["Build SSM payload<br/> SSMExtraSection.dat"]
  end

  %% Message Bus
  subgraph ServiceBus
    SB["Optional File Move and Service Bus message<br>if breaking SP_PUBS_SSM_ExtraSections into two instances "]
  end

  subgraph DIR["/aa-pubs-extrasection"]
    FA["Extra_Section_Schedule_File_20260122_010031.txt"]
  end

  subgraph DIRT["/aa-pubs-extrasection"]
    FB["SSMExtraSection_20260122_010031.dat"]
  end

  %% Message Bus
  subgraph NPPortal
    FT["<pre>aa-pubs-transmit
- manifest.dat.20260122085603
- SSMExtraSection_20260122_010031.dat+ITASSM+3084383
- PUBS[ITASSM]
- /mnt/Transfer/PUBS/ITASSM/Outbound</pre> "]
  end

  %% Consumer Application Nodes
  subgraph EMFT
    C1["New Configuration for SSM"]

  end

  %% External system
  EXT["ITA/Google processes SSM"]

  %% Edges
  G3 --> SB
  G4 --> FA
  SB --> G8
  G8 --> FT
  G8 --> FB
  FT --> C1
  C1 --> EXT

```


## Discussion Points

- ** We will need ITA contact/interface for developing the SFTP formats and validation. **
- Will ITA have an error reporting system?
- When will we get our SabreMQueue?  Do we own it? Is it ours?

### This is an example of an extra section from the SabreMQ in Casper's json message formatting.

```json
{
  "messageId": 375698847,
  "ssimType": "ADD",
  "effectiveDate": "Jan 10, 2026",
  "discontinueDate": "Jan 10, 2026",
  "flightNumber": 9603,
  "messageText": "A AA 9603  01J10JAN2610JAN26     6  BOG09000900-0500  MIA13001300-0500  7M8CJRDIUYBHKMLGVSNQOET     XX                 II                                                               E210F2A317BAF2094 AA 9603  01J                801BOGMIAMAX                                                                                                                                              E210F2A317BBB8094 AA 9603  01J                109BOGMIAV V V V V V V V V V V V V V V V V V V V                                                                                                          E210F2A317BBBC094 AA 9603  01J                106BOGMIAC016J016R014D012I008U002Y156B148H136K123M103L090G061V047S039N031Q027O023E008T003                                                                 E210F2A317BBC6095 AA        9603                                                                                                                                                                        E210F2A317DDBC09",
  "clockTime": 16289786634490802000,
  "msgReceivedTime": "2026-01-09 19:51:30.424"
}
```
<br>
A<br> AA 9603  01J10JAN2610JAN26     6  BOG09000900-0500  MIA13001300-0500  7M8CJRDIUYBHKMLGVSNQOET     XX                 II                                                               E210F2A317BAF2094<br> AA 9603  01J                801BOGMIAMAX                                                                                                                                              E210F2A317BBB8094<br> AA 9603  01J                109BOGMIAV V V V V V V V V V V V V V V V V V V V                                                                                                          E210F2A317BBBC094<br AA 9603  01J                106BOGMIAC016J016R014D012I008U002Y156B148H136K123M103L090G061V047S039N031Q027O023E008T003                                                                 E210F2A317BBC6095 AA        9603                                                                                                                                                                        E210F2A317DDBC09
<br>
<br\><br\>

MORE JSON EXAMPLES:

```json
{"messageId":377050188,"ssimType":"ADD","effectiveDate":"Jan 22, 2026","discontinueDate":"Jan 22, 2026","flightNumber":9602,"messageText":"A AA 9602  01J22JAN2622JAN26   4    EZE20452045-0300  JFK05300530-0500  772XX                       XX                 II                                                               E2200FAA3C0196054 AA 9602  01J                801EZEJFKPE8                                                                                                                                              E2200FAA3C0260054 AA 9602  01J                106EZEJFKCJRDIUWPXYBHKMLGVSNQOET                                                                                                                          E2200FAA3C0264054 AA 9602  01J                109EZEJFKV V V V V V V V V V V V V V V V V V V V V V V                                                                                                    E2200FAA3C0268054 AA 9602  01J                106EZEJFKC035J035R032D026I018U004W024P019X002Y212B201H184K167M140L123G083V064S053N042Q036O032E021T011                                                     E2200FAA3C0274055 AA        9602                                                                                                                                                                        E2200FAA3C2EA805","clockTime":16294040675652048389,"msgReceivedTime":"2026-01-21 20:21:15.435"}
```
```json
{"messageId":377050170,"ssimType":"ADD","effectiveDate":"Jan 19, 2026","discontinueDate":"Jan 19, 2026","flightNumber":9607,"messageText":"A AA 9607  01J19JAN2619JAN261       LAX17301730-0800  LHR12051205+0000  77WXX                       XX                 II                                                               E21D530DA054AA094 AA 9607  01J                801LAXLHRPE9                                                                                                                                              E21D530DA0555E094 AA 9607  01J                106LAXLHRFAZCJRDIUWPXYBHKMLGVSNQOET                                                                                                                       E21D530DA05562094 AA 9607  01J                109LAXLHRB V V V V V V V V V V V V V V V V V V V V V V V V V                                                                                              E21D530DA05568094 AA 9607  01J                106LAXLHRF008A007Z005C052J052R047D039I026U005W028P022X003Y216B205H188K171M143L125G084V065S054N043Q037O032E022T011                                         E21D530DA0559C095 AA        9607                                                                                                                                                                        E21D530DA083F009","clockTime":16293270344885905929,"msgReceivedTime":"2026-01-19 16:06:46.402"}
{"messageId":377050171,"ssimType":"ADD","effectiveDate":"Jan 20, 2026","discontinueDate":"Jan 20, 2026","flightNumber":9601,"messageText":"A AA 9601  01J20JAN2620JAN26 2      LHR17351735+0000  JFK18351835-0500  77WXX                       XX                 II                                                               E21D53916B1362054 AA 9601  01J                801LHRJFKPE9                                                                                                                                              E21D53916B13F6054 AA 9601  01J                106LHRJFKFAZCJRDIUWPXYBHKMLGVSNQOET                                                                                                                       E21D53916B13FA054 AA 9601  01J                109LHRJFKB V V V V V V V V V V V V V V V V V V V V V V V V V                                                                                              E21D53916B1416054 AA 9601  01J                106LHRJFKF008A007Z005C052J052R047D039I026U005W028P022X003Y216B205H188K171M143L125G084V065S054N043Q037O032E022T011                                         E21D53916B1424055 AA        9601                                                                                                                                                                        E21D53916B3E7C05","clockTime":16293270910928118277,"msgReceivedTime":"2026-01-19 16:09:03.620"}
{"messageId":377050063,"ssimType":"ADD","effectiveDate":"Jan 19, 2026","discontinueDate":"Jan 19, 2026","flightNumber":9608,"messageText":"A AA 9608  01J19JAN2619JAN261       LAX17301730-0800  LHR12051205+0000  788XX                       XX                 II                                                               E21D641874D94C014 AA 9608  01J                801LAXLHR78W                                                                                                                                              E21D641874D9D6014 AA 9608  01J                106LAXLHRCJRDIUWPXYBHKMLGVSNQOET                                                                                                                          E21D641874D9DA014 AA 9608  01J                109LAXLHRV V V V V V V V V V V V V V V V V V V V V V V                                                                                                    E21D641874D9DE014 AA 9608  01J                106LAXLHRC020J020R018D015I010U002W028P023X003Y186B177H162K147M123L108G073V056S047N038Q032O028E019T010                                                     E21D641874D9EA015 AA        9608                                                                                                                                                                        E21D641874F7E801","clockTime":16293289083098713089,"msgReceivedTime":"2026-01-19 17:23:00.636"}
{"messageId":377050064,"ssimType":"ADD","effectiveDate":"Jan 20, 2026","discontinueDate":"Jan 20, 2026","flightNumber":9602,"messageText":"A AA 9602  01J20JAN2620JAN26 2      LHR15351535+0000  JFK18401840-0500  788XX                       XX                 II                                                               E21D64B3E97F3A084 AA 9602  01J                801LHRJFK78W                                                                                                                                              E21D64B3E97FE8084 AA 9602  01J                106LHRJFKCJRDIUWPXYBHKMLGVSNQOET                                                                                                                          E21D64B3E97FEC084 AA 9602  01J                109LHRJFKV V V V V V V V V V V V V V V V V V V V V V V                                                                                                    E21D64B3E97FF2084 AA 9602  01J                106LHRJFKC020J020R018D015I010U002W028P023X003Y186B177H162K147M123L108G073V056S047N038Q032O028E019T010                                                     E21D64B3E97FFE085 AA        9602                                                                                                                                                                        E21D64B3E9A12408","clockTime":16293289750775675400,"msgReceivedTime":"2026-01-19 17:25:43.211"}

```
```json
```
```json
```

[Schedule File Processor application:](https://github.com/AAInternal/FltInvhub_Schedule_FileProcessor/)

[Business Rules wiki:](https://github.com/AAInternal/FltInvhub_Schedule_FileProcessor/wiki/File-Processor-Business-Rules)



{"messageId":377050188,"ssimType":"ADD","effectiveDate":"Jan 22, 2026","discontinueDate":"Jan 22, 2026","flightNumber":9602,"messageText":"A AA 9602  01J22JAN2622JAN26   4    EZE20452045-0300  JFK05300530-0500  772XX                       XX                 II                                                               E2200FAA3C0196054 AA 9602  01J                801EZEJFKPE8                                                                                                                                              E2200FAA3C0260054 AA 9602  01J                106EZEJFKCJRDIUWPXYBHKMLGVSNQOET                                                                                                                          E2200FAA3C0264054 AA 9602  01J                109EZEJFKV V V V V V V V V V V V V V V V V V V V V V V                                                                                                    E2200FAA3C0268054 AA 9602  01J                106EZEJFKC035J035R032D026I018U004W024P019X002Y212B201H184K167M140L123G083V064S053N042Q036O032E021T011                                                     E2200FAA3C0274055 AA        9602                                                                                                                                                                        E2200FAA3C2EA805","clockTime":16294040675652048389,"msgReceivedTime":"2026-01-21 20:21:15.435"}
