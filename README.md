# HL7 v2 Integration Harness
## Overview
This project is a self-directed healthcare integration simulator designed to model real-world HL7 v2 message flows between clinical systems using MLLP transport and an interface engine.

It demonstrates end-to-end ADT messaging patterns commonly found in hospital environments, including:

* HL7 v2 message construction and transmission

* MLLP socket communication

* ACK correlation (AA / AE / AR)

* retry and backoff logic

* error classification

* message archiving

* Dockerized NextGen Connect (Mirth) channels

The goal is to emulate how upstream registration systems interact with downstream interface engines in production healthcare environments.

This project is intended to showcase practical healthcare interoperability concepts rather than serve as a production-ready framework.

---
## What This Simulates (Clinical Context)

In a typical hospital workflow:

1. A registration system generates an ADT message (patient admission / update).

2. The message is transmitted over MLLP.

3. An interface engine (such as Mirth) receives the message.

4. An HL7 ACK is returned.

5. Messages are archived or flagged depending on success or failure.

This harness reproduces that pattern:

* A Java-based HL7 sender generates ADT messages using HAPI.

* Messages are framed and transmitted via MLLP.

* A Dockerized NextGen Connect (Mirth) instance listens for inbound messages.

* ACK responses are parsed and correlated.

* Failed transmissions are retried with backoff.

* Messages are classified and archived based on outcome.

This mirrors common operational realities in healthcare integration teams.
---
## Architecture (High Level)
```
[ Java HL7 Sender ]
        |
        |  HL7 v2 over MLLP
        v
[ Mirth Listener (Docker) ]
        |
        v
   ACK Response
```
---
## Features

* HL7 v2 ADT message generation

* MLLP framing and socket communication

* ACK control ID correlation

* Retry / backoff on failed delivery

* Error classification (AA / AE / AR)

* Message archiving for audit/debugging

* Docker Compose environment for reproducible setup

* Simple logging for troubleshooting
---
Tech Stack

* Java (HAPI HL7)

* Maven

* Docker / Docker Compose

* NextGen Connect (Mirth)

* TCP / MLLP
---
## Getting Started
### Prerequisites

* Docker + Docker Compose

* Java 11+

* Maven
---
## Start Mirth
```
docker-compose up -d
```
This launches a local Mirth instance listening for HL7 traffic.

---
## Build and Run Sender
```
mvn clean package
java -jar target/hl7-sender.jar
```
The sender will generate ADT messages and transmit them to the Mirth listener.

ACK responses are logged and messages are archived based on outcome.

---
## Sample HL7(Exerpt)
```
MSH|^~\&|ADT|HOSPITAL|MIRTH|INTERFACE|20260101||ADT^A01|12345|P|2.5
PID|||123456||Doe^John
PV1||I
```

---
## What This Demonstrates
This project reflects common responsibilities of Interface / Integration Analysts:

* building and testing HL7 pipelines

* validating ACK behavior

* troubleshooting transport failures

* understanding clinical workflows behind ADT messages

* working with interface engines

* managing retries and error paths

* creating reproducible environments

It was built as hands-on practice for healthcare integration roles involving HL7, MLLP, and Mirth.

---
## Future Enhancements
* ORM / ORU message support

* TLS transport

* database-backed message persistence

* richer Mirth channel logic

* metrics dashboard

---
## Author

Glen Tanner

Healthcare Systems/Integration

------------------------------------------------------------------------

## 📜 License

MIT Copyright &copy; 2025 Glen Tanner

------------------------------------------------------------------------
