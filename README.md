# Online Examination Network (SDN + TCP Sockets)

SDN-Socket Programming Jackfruit Mini Project, Topic #18.

A multi-user online examination system built on TCP sockets and deployed in a Mininet network managed by an SDN controller. The controller monitors and manages exam traffic, and the project compares its behavior against a plain learning-switch baseline under congestion.

## Team

| Name | SRN | Role |
|---|---|---|
| Poornaprajna | PES2UG25CS372 |  Application (server, client, protocol) |
| Rampranav | PES2UG25CS359 | SDN and network (topology, controller) |
| Pragna Adnoor | PES2UG25CS377 | Integration, experiments, documentation |

## Table of contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Repository structure and ownership](#3-repository-structure-and-ownership)
4. [Requirements and installation](#4-requirements-and-installation)
5. [Configuration](#5-configuration)
6. [How to run](#6-how-to-run)
7. [Application protocol](#7-application-protocol)
8. [SDN design](#8-sdn-design)
9. [Test cases](#9-test-cases)
10. [Experiments and results](#10-experiments-and-results)
11. [Limitations and future work](#11-limitations-and-future-work)
12. [Contributing rules](#12-contributing-rules)
13. [Documentation index](#13-documentation-index)

## 1. Overview

**Problem.** Develop a multi-user online examination application where students connect through TCP sockets. The SDN network monitors and manages the examination traffic.

**What the system does**

- Concurrent TCP exam server that handles many students at once
- Question delivery, answer submission, server-side exam timer, and result calculation
- Mininet topology with a bottleneck link (and an alternate path) to create realistic congestion
- SDN controller that classifies exam traffic, monitors flow and port statistics, and reacts dynamically (prioritization, rate limiting, rerouting)
- Experiments comparing the SDN-managed network against a static baseline

## 2. Architecture

_Insert the architecture diagram here (`docs/architecture.png`)._

```
 Student clients (h2..hN)  --TCP-->  Switches (OpenFlow)  -->  Exam server (h1)
                                          |   ^
                                    flow rules / stats
                                          v   |
                                    SDN controller  <--exam events--  Exam server
```

**Components**

| Component | Description |
|---|---|
| Exam server | Concurrent TCP server: login, questions, answers, timer, scoring |
| Student clients | Interactive client and an automated client that simulates N students |
| Mininet topology | Hosts, OpenFlow switches, bottleneck link, alternate path |
| SDN controller | Classification, stats polling, dynamic policies, receives exam events |
| Integration module | Lets the server notify the controller of exam events |
| Experiment scripts | Generate load, run baseline and SDN modes, analyze results |

**Socket-SDN integration.** The server notifies the controller of exam events (exam start, exam end, student count changes). The controller responds by installing, adjusting or removing flow rules.

## 3. Repository structure and ownership

Every file has exactly one owner. Only the owner edits it.

```
.
├── README.md                  # C
├── config.json                # C (A and B read only)
├── requirements.txt           # C
├── .gitignore                 # C
├── .github/CODEOWNERS         # C
├── server/                    # A: server.py, exam_logic.py, questions.json
├── client/                    # A: client.py, auto_client.py
├── tests/functional/          # A
├── controller/                # B: baseline_controller.py, exam_controller.py,
│                              #    stats_monitor.py, rest_api.py
├── topology/                  # B: topo.py
├── integration/               # C: notifier.py
├── experiments/               # C: run_experiment.sh, traffic_gen.sh, analyze.py
├── results/                   # C: raw/ and graphs/
├── logs/                      # runtime output (git-ignored)
└── docs/
    ├── protocol.md            # A
    ├── sdn_design.md          # B
    ├── architecture.md        # C
    └── test_cases.md          # C
```

## 4. Requirements and installation

**Environment:** Ubuntu 20.04 / 22.04 VM (VirtualBox or VMware) with sudo access. Mininet runs only on Linux. The application code (server and client) can also be tested on any OS using localhost.

```bash
# System packages
sudo apt update && sudo apt upgrade -y
sudo apt install mininet openvswitch-switch iperf python3-pip git -y

# Clone the repository
git clone <repo-url>
cd <repo-name>

# Python dependencies (controller, plotting, etc.)
pip3 install -r requirements.txt
```

**Verify Mininet**

```bash
sudo mn --topo single,3
mininet> pingall      # expect 0% packet loss
mininet> exit
sudo mn -c            # clean up
```

**Verify the controller**

```bash
ryu-manager --version
```

> The controller is assumed to be Ryu (OpenFlow 1.3). Update this section if the team uses a different controller.

## 5. Configuration

Shared settings live in `config.json`. Do not hardcode IPs or ports in code.

| Key | Meaning | Draft value |
|---|---|---|
| `server_ip` | Exam server address inside Mininet | `10.0.0.1` |
| `server_port` | Exam server TCP port | `5000` |
| `controller_ip` | Address the exam server uses to reach the controller's event endpoint | _set after topology is final_ |
| `controller_rest_port` | Port of the controller's event endpoint | `8080` |

**Notes on these values**

- `server_ip` assumes the server runs on `h1` with Mininet's default addressing. The real mapping is in `docs/sdn_design.md`.
- Each Mininet host has its own network namespace, so `127.0.0.1` inside `h1` is **not** the controller. `controller_ip` must be an address the hosts can reach (for example a NAT or root-namespace interface added in `topology/topo.py`). It is finalized once the topology is final.
- On macOS, port 5000 is often used by AirPlay Receiver. If local testing fails, use another port (for example 5050) and keep `config.json` consistent.
- Port 8080 is the default of Ryu's built-in web server. Change it if another tool uses it.

## 6. How to run

Run all commands from the repository root. Open one terminal per step, in this order.

**Step 1: clean old Mininet state**

```bash
sudo mn -c
```

**Step 2: start the controller** (Terminal 1)

```bash
# SDN mode
ryu-manager controller/exam_controller.py

# Baseline mode (plain learning switch)
ryu-manager controller/baseline_controller.py
```

**Step 3: start the topology** (Terminal 2)

```bash
sudo python3 topology/topo.py --mode sdn          # or --mode baseline
```

This opens the Mininet CLI. The host mapping (`h1` = exam server, `h2` to `hN` = students, remaining hosts = background traffic) is documented in `docs/sdn_design.md`.

**Step 4: start the exam server** (inside the Mininet CLI)

```bash
mininet> h1 python3 server/server.py --port 5000 &
```

**Step 5: start background traffic** (optional, to create congestion)

```bash
mininet> h8 bash experiments/traffic_gen.sh --level heavy &
```

**Step 6: start student clients**

```bash
# Automated: simulate 5 students
mininet> h2 python3 client/auto_client.py --host 10.0.0.1 --port 5000 --students 5 &

# Interactive: a single student
mininet> h3 python3 client/client.py --host 10.0.0.1 --port 5000
```

**Run a full automated experiment (one command)**

```bash
bash experiments/run_experiment.sh --mode sdn --students 10 --load heavy
bash experiments/run_experiment.sh --mode baseline --students 10 --load heavy
python3 experiments/analyze.py results/raw/
```

**Run the application locally (outside Mininet) for quick testing**

```bash
python3 server/server.py --port 5000
python3 client/client.py --host 127.0.0.1 --port 5000
```

## 7. Application protocol

Full specification: [`docs/protocol.md`](docs/protocol.md).

| Message | Direction | Purpose |
|---|---|---|
| `LOGIN` | client to server | Join the exam with a student ID |
| `START` | server to client | Begin the exam (duration, number of questions) |
| `QUESTION` | server to client | Deliver a question |
| `ANSWER` | client to server | Submit an answer |
| `ACK` | server to client | Confirm receipt |
| `TIME_UP` | server to client | Exam timer expired |
| `RESULT` | server to client | Final score |

_Update this table so it matches `docs/protocol.md` exactly._

## 8. SDN design

Full details: [`docs/sdn_design.md`](docs/sdn_design.md).

- **Classification:** TCP traffic to the exam server port is matched and given higher priority than background traffic.
- **Monitoring:** flow and port statistics are polled periodically to compute link utilization and per-student throughput.
- **Dynamic behavior:** _(fill in what you implemented)_ prioritization under congestion, limiting background flows, rerouting on link failure.
- **Socket-SDN integration:** exam events from the server trigger controller actions.

| Parameter | Value |
|---|---|
| Stats polling interval | _x_ seconds |
| Congestion threshold | _x_% link utilization |
| Priority of exam flows | _x_ |

## 9. Test cases

Full table: [`docs/test_cases.md`](docs/test_cases.md).

| ID | Scenario | Expected result | Status |
|---|---|---|---|
| T1 | Normal exam with one student | Questions delivered, score correct | |
| T2 | Multiple concurrent students | All complete, no mixed-up answers | |
| T3 | Late submission after timer | Answer rejected | |
| T4 | Client disconnects mid-exam | Server stays up, session handled | |
| T5 | Duplicate login | Second login rejected | |
| T6 | Exam under heavy congestion | Exam traffic prioritized (SDN mode) | |
| T7 | Link failure | Traffic rerouted, exam continues | |

## 10. Experiments and results

**Metrics:** response time per question, answer-submission latency, throughput, packet loss, link utilization, fairness across students (Jain's index).

**Conditions:** baseline vs SDN, with 3 / 6 / 10 students and no / medium / heavy congestion.

_Insert graphs from `results/graphs/` here, for example:_

```
![Response time: baseline vs SDN](results/graphs/response_time.png)
```

_Add a short analysis: what improved, by how much, and why._

## 11. Limitations and future work

- _List known limitations, for example a simulated rather than real network, no encryption, a small question bank._
- _List ideas for future work._

## 12. Contributing rules

1. Edit only the files you own (section 3). Need a change elsewhere? Open an Issue or message the owner.
2. Never commit directly to `main`. Use one branch per task (`a/...`, `b/...`, `c/...`) and open a pull request.
3. Every pull request is reviewed by another member (A's by B, B's by C, C's by A).
4. Before pushing, run `git pull --rebase origin main`. Never force-push.
5. Do not commit generated files (`__pycache__`, `logs/`, large pcaps, virtual environments).
6. Read shared settings from `config.json`. Do not hardcode IPs or ports.
7. Run Python from the repository root so cross-folder imports work.
8. Merge small and often. Large late merges cause conflicts and demo failures.

## 13. Documentation index

| Document | Contents |
|---|---|
| [`docs/protocol.md`](docs/protocol.md) | Message formats and error codes |
| [`docs/sdn_design.md`](docs/sdn_design.md) | Flow rules, policies, thresholds, host/IP mapping |
| [`docs/architecture.md`](docs/architecture.md) | Architecture diagram and communication flow |
| [`docs/test_cases.md`](docs/test_cases.md) | Test cases and results |
