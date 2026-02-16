
# Runbook: HL7 v2 Integration Harness (Java) + Dockerized Mirth

This runbook provides a repeatable, step-by-step process to start the Dockerized NextGen Connect (Mirth) environment, deploy the inbound MLLP channel using **Mirth Administrator (desktop app)**, and run the Java sender to verify end-to-end HL7 v2 + ACK behavior.

---

## Scope


- Starts Mirth via Docker Compose
- Connects using Mirth Administrator (desktop client)
- Imports + deploys the channel XML
- Verifies MLLP listener availability
- Builds and runs the Java harness
- Confirms ACK correlation and success
- Provides fast troubleshooting checks

---

## Assumptions / Defaults

- Repo root contains: `docker-compose.yml`, `pom.xml`, `mirth/`, `src/`
- Mirth Admin port: `8443` (HTTPS)
- MLLP listener port: `2575` (TCP)
- Default credentials on first run: `admin / admin`
- Channel XML: `mirth/channels/LLP_Inbound_2575.xml`

If your environment differs, update the values accordingly.

---

## 0) Hard Reset (Stop Everything)

From repo root:

```bash
docker compose down
```

## 1) Start Mirth in Docker

```bash
docker compose up -d
```

Confirm container + ports
```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

You should see port mappings for 8443 and 2575

* 0.0.0.0:8443 -> 8443/tcp
* 0.0.0.0:2575 -> 2575/tcp

## 2) Wait Until Mirth Is Ready
Tail logs until you see startup completion messages:
```bash
docker logs -f mirth
```
Stop tailing with CTRL+C once Mirth indicates it is running.

## 3) Launch Mirth Admin
Run you local Administrator client
```bash
$mirth-administrator
```
Connect settings
* Server: localhost
* Port: 8443
* Use SSL: enabled
* Username: admin
* Password: admin

If prompted to trust the certificate, accept

## 4) Import the Channel XML
In Mirth Administrator:
    1. Channels -> Import
    2. Select: mirth/channels/LLP_Inbound_2575.xml
    3. Complete import
Verify the channel configuration:
* Source Connnector: LLP Listener / TCP (MLLP) on port 2575
* Destination: File Writer (archive) or as configured in the XML

## 5) Deploy the Channel
In Mirth Administrator:
* Channels -> select the channel -> Deploy
Confirm it shows as Deployed (and not errored) in the Dashboard.

## 6) Verify the MLLP Port is Reachable
From the terminal:
```bash
nc -vz 127.0.0.1 2575
```
Expected: connection succeeds / port open
If this fails, the channel is not deployed, the listener port differs, or Docker ports are not mapped.

## 7) Build the Java Harness
From repo root:
``` bash
mvn clean package
ls -lh target/*.jar
```
Identify the jar produced (example):
* target/hl7-v2-integration-harness-0.1.0.jar

## 8) Run the Harness and Confirm ACK Success
```bash
java -jar target/hl7-v2-integration-harness-0.1.0.jar
```
Expected log pattern:
* Sending ADT^A01 ...
* ACK msaCode=AA ... correlated=true
* SUCCESS

This confirms end-to-end:
* MLLP framing over TCP
* Message acceptance by Mirth
* ACK creation + correlation (MSH-10 <-> MSA-2)

## 9) Quick Troubleshooting Checklist
## A) Port conflict/container already exists

Symptoms:
* "container name/mirth is already in use"
Fix:
```bash
docker compose down
docker rm -f mirth
docker compose up -d
```
## B) JAR not found/"Unable to access jarfile"
Fix:
```bash
mvn clean package
ls -lh target
java -jar target/<exact-jar-name>.jar
```
## C) MLLP connection refused/cannot connect 2575
Check:
* Channel is imported and deployed
* Docker port mapping includes 2575:2575

Commands:
```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
nc -vz 127.0.0.1 2575
```

## D) Received ACK but parsing shows NO_MSA/contolled mismatch
Likely causes:
* Channel not producing a standard HL7 ACK
* Response contains non-HL7 text(rare, but possible if misrouted)
* ACK parsing expects different line endings

Checks:
* In mirth Administrator -> Message Browser, open the received message and inspect raw response/ACK
* Confirm the outbound is a proper HL7 ACK with an MSA segment

## 10) Shutdown
```bash
docker compose down
```

```bash
git add RUNBOOK.md
git commit -m "Add runbook for Dockerized Mirth + harness execution"
git push
```


## What happens message-wise (the real explanation)
### A) The sender constructs an HL7 v2 ADT^A01

* It builds an HL7 message string with segments like:

    * MSH (header)

    * PID (patient)

    * PV1 (visit)

#### Key field

* MSH-10 = Message Control ID
This is the unique ID used to correlate the ACK.

### B) The sender opens a TCP connection to Mirth (MLLP)

* Host: 127.0.0.1

* Port: 2575

Then it wraps the HL7 payload using MLLP framing:

* Start byte: 0x0B

* End bytes: 0x1C 0x0D

This is how HL7 v2 is commonly transmitted over raw TCP in real hospital interfaces.

### C) Mirth receives the message and validates it

Inside the channel:

* Mirth parses the inbound HL7

* Applies the filter rules (ex: “PID must exist”)

If it passes:

* Mirth generates an ACK (AA)

If it fails:

* Mirth generates an ACK (AE or AR) depending on configuration

### D) The sender reads the ACK and correlates it

The ACK includes:

* MSA-1 = acknowledgment code (AA/AE/AR)

* MSA-2 = message control ID being acknowledged

#### Critical correlation rule

* MSH-10 (original message) must equal MSA-2 (ACK’s message ID)

That proves the ACK is for this message, not some other message.

### E) Sender behavior based on ACK code
#### If AA (Accept)

* Mark success
* Archive payload (optional) and exit cleanly

#### If AE (Error)

* Treat as retryable (depending on your rules)
* Retry with backoff up to max attempts

#### If AR (Reject)

* Treat as non-retryable
* Archive payload + ACK for audit
* Fail fast (no point hammering a bad message)

This mirrors real interface operations: rejects are remediated, not retried.

### F) File archival in Mirth

On the Mirth side, the File Writer writes received messages to disk.
This shows the operational reality:

* interfaces often keep a record for audit
* support teams replay messages after upstream fixes

### How to “prove” it worked (what you point to)

1. Sender logs

Shows:

    *sending
    * ACK code
    * correlated=true
    * success

2. Mirth Dashboard

    * message counts increment (Received/Sent/Error)

3. Message Browser

* shows the inbound HL7 message and status

4. Archive output

* files written in your out/ or Mirth destination folder

## 30-second “explain the whole thing” version (use in interviews)

> “I bring up a dockerized Mirth instance, deploy an LLP listener channel on port 2575, then run a Java harness that constructs HL7 v2 ADT messages, sends them over MLLP/TCP, reads the ACK, and validates correlation by matching MSH-10 with MSA-2. Based on AA/AE/AR it applies retry rules or rejects and archives payloads for audit and replay. It’s a simulated clinical interface implementation focused on failure modes and recoverability.”















































































