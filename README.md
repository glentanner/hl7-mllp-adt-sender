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

Java (HAPI HL7)

Maven

Docker / Docker Compose

NextGen Connect (Mirth)

TCP / MLLP

Getting Started
Prerequisites

Docker + Docker Compose

Java 11+

Maven
---

## 🏥 Clinical Context

This harness models a simplified upstream **patient registration system** sending ADT messages to a downstream **interface engine**, mirroring common hospital integration patterns.

The focus is on:
- Correct sequencing and acknowledgment of patient events
- Detecting and classifying malformed clinical data
- Ensuring failed messages are traceable and recoverable
- Preventing silent data loss in regulated environments

These concerns are foundational to clinical systems such as pharmacy, laboratory, oncology, and downstream EMR integrations.
While this harness runs in a non-production environment, the behaviors modeled reflect production interface patterns commonly encountered in hospital IT operations.

---

## 🧱 Logical Architecture (Non-Production)

    +------------------+        MLLP/TCP        +---------------------------+
    |  Java Sender     |  ------------------>  |  NextGen Connect (Mirth)  |
    |  (ADT Producer)  |  <------------------  |  TCP Listener + ACK       |
    +------------------+        HL7 ACK         +---------------------------+
                                                  |
                                                  v
                                          File Writer (archive)

------------------------------------------------------------------------

## 🛠️ Tech Stack

- Java 25 (compiled for Java 21)
- Maven
- Docker & Docker Compose
- NextGen Connect (Mirth) 4.5.2
- Ubuntu 24.04
- IntelliJ IDEA

---

## 📋 Prerequisites

- Java 21+
- Maven 3.9+
- Docker & Docker Compose
- Git

---

## 🚀 Quick Start

### 1️⃣ Start Mirth in Docker

From the repo root:

```bash
docker compose up -d
```
------------------------------------------------------------------------

Services exposed:

Admin UI: https://localhost:8443

Web UI: http://localhost:8080

MLLP Listener: port 2575

Default login on first run:\
`admin / admin`

------------------------------------------------------------------------

### 2️⃣ Import the Mirth Channel

A preconfigured NextGen Connect (Mirth) channel is included.

In **Mirth Connect Administrator**:
1. Channels → Import
2. Import:
   `mirth/channels/LLP_Inbound_2575.xml`
3. Deploy the channel

The channel includes:
- TCP Listener (MLLP server) on port 2575
- Source Filter enforcing required PID segment
- Auto-generated ACKs (AA / AR)
- File Writer archiving inbound HL7


------------------------------------------------------------------------

### 3️⃣ Build and Run the Java Sender

``` bash
mvn clean package
java -jar target/hl7-v2-integration-harness-0.1.0.jar
```

------------------------------------------------------------------------

### 4️⃣ Verify Success

Expected logs:

    Sending ADT^A01 attempt=1 controlId=...
    ACK msaCode=AA msaControlId=... correlated=true latencyMs=...
    SUCCESS

Archived messages appear in:

``` bash
out/
  inbound_<timestamp>.hl7
```

Containing the HL7 message.

------------------------------------------------------------------------

## 📄 Example HL7 Message

    MSH|^~\&|JAVA_SENDER|HOSP|MIRTH|HOSP|20251219152533||ADT^A01|<uuid>|P|2.3
    PID|1||123456^^^HOSP^MR||DOE^JANE||19800101|F
    PV1|1|E|ER^01^01^HOSP|||||||||||||||V0001


------------------------------------------------------------------------

## ⚙️ Configuration

Environment variables:

  Variable                Default       Description
  ----------------------- ------------- ----------------
  `HL7_HOST`              `127.0.0.1`   MLLP host
  `HL7_PORT`              `2575`        MLLP port
  `HL7_READ_TIMEOUT_MS`   `5000`        ACK timeout
  `HL7_MAX_ATTEMPTS`      `3`           Retry attempts
  `HL7_BACKOFF_MS`        `500`         Retry delay

Example:

``` bash
HL7_HOST=127.0.0.1 HL7_PORT=2575 \
java -jar target/hl7-v2-integration-harness-0.1.0.jar
```

------------------------------------------------------------------------

## 🧠 Key Concepts

-   **MLLP Framing**\
    Messages are wrapped with:

    -   Start: `0x0B`
    -   End: `0x1C 0x0D`

-   **ACK Correlation**\
    `MSH-10` (control ID) must match `MSA-2` in the ACK.

-   **AA / AE / AR**\
    Accept, Error, Reject --- determines sender behavior.
    
## ❌ Negative ACK Handling

- Messages missing required segments (e.g. PID) are **rejected by Mirth**
    - Messages missing required segments (e.g., PID) are rejected
    - AR responses are not retried
    - Failed payloads and ACKs are archived for inspection

    (This mirrors real-world interface engine behavior)

------------------------------------------------------------------------

## 🗂️ Repo Structure

    .
    ├── src/main/java/com/medlydesign/hl7mllp
    │   ├── Main.java
    │   ├── MllpClient.java
    │   ├── Hl7AdtBuilder.java
    │   ├── Hl7Ack.java
    │   └── Hl7AckParser.java
    ├── docker-compose.yml
    ├── out/               # HL7 archives (gitignored)
    ├── pom.xml
    └── README.md

------------------------------------------------------------------------

## 🏁 Planned Clinical Scenarios

### ADT lifecycle progression (A01 → A08 → A03)
### Clinical order workflows (ORM) and downstream result flows (ORU)
### Sequencing and dependency handling common in medication and oncology workflows

------------------------------------------------------------------------

## 📜 License

MIT Copyright &copy; 2025 Glen Tanner

------------------------------------------------------------------------

## 👤 Author

Built by Glen Tanner as a self-directed integration project exploring HL7 v2 messaging, MLLP transport behavior, failure handling, and workflow progression in regulated healthcare systems.
