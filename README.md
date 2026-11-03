# SDN Network Monitoring and Alert System

**Course:** Computer Networks (CN) Project · SDN-Socket Programming
**Problem statment** Network Monitoring and Alert System · MEMBER 1: Kalmesh Ningappa Bisankoppa, MEMBER 2: Jayanth Chowdary, MEMBER 3: Jeevitha V

## Project Documents

| Document | Description |
|---|---|
| [Architecture and Design](docs/ARCHITECTURE.md) | Topology, components, data flow, alert logic |
| [Team Contributions](CONTRIBUTIONS.md) | Work done by each team member |
| [Screenshots](SCREENSHOTS.md) | Results and functionality |

## 1. Objective

Build a distributed network-monitoring system on a Mininet network controlled by an
OpenFlow (SDN) controller:

- Hosts send **UDP telemetry** to a central monitoring server (low-level socket programming).
- The **SDN controller** collects switch statistics (port and flow counters), computes link
  utilization, and combines them with the application telemetry to detect abnormal
  conditions.
- The controller raises **alerts** and reacts dynamically (drops a flow that saturates a link).
- A **web dashboard** (Python only) shows the network state live and lets you trigger test
  scenarios.

## 2. Features

- UDP telemetry agents on h1-h3 sending a message every 5 s
- Central monitoring server receiving UDP on `10.0.0.100:5000`
- Per-flow OpenFlow rules installed by the controller
- Link utilization from port statistics
- Alerts for: link utilization (WARNING / CRITICAL), silent host, telemetry flood, loss after the switch
- Automatic mitigation: drop rule for the heaviest non-telemetry flow on CRITICAL
- Live web dashboard with scenario buttons
- Log analysis script to compare runs (mitigation on vs off)

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full design.

## 3. Requirements

- Ubuntu with **Mininet** and **Open vSwitch**
- `iperf` (traffic generator for the congestion scenario)
- Python 3.11 environment with **os-ken 2.8.1** for the controller

```bash
sudo apt install -y openvswitch-switch iperf
curl -LsSf https://astral.sh/uv/install.sh | sh && source ~/.local/bin/env
uv venv --python 3.11 ~/ctrl-env
source ~/ctrl-env/bin/activate
uv pip install "os-ken==2.8.1" "eventlet==0.33.3"
deactivate
```

## 4. How to Run

```bash
sudo mn -c
sudo python3 run_all.py            # then open http://localhost:8080
```

| Option | Meaning |
|---|---|
| `--cli` | also open the Mininet prompt |
| `--mitigate 0` | disable the automatic drop (for the comparison run) |
| `--bw 10` | link speed in Mbit/s (default 10) |
| `--osken PATH` | path to `osken-manager` if not in `~/ctrl-env/bin` |
| `--server-script PATH` | server script (default `server.py` in the same folder) |

Manual mode: run `osken-manager monitor_ctrl.py` in one terminal and
`sudo python3 part_b_topology.py --start-app` in another. `--demo` runs automated checks and
saves evidence in `evidence/`.

## 5. Test Scenarios (dashboard buttons)

| Scenario | Action | Expected result |
|---|---|---|
| Normal | none | all hosts OK, utilization about 0% |
| Congestion | Flood link (9 Mb/s UDP, 30 s) | link bars red, `CRITICAL`, then `MITIGATION` |
| Telemetry flood | Telemetry flood from h1 | `FLOOD` status and alert |
| Host failure | Stop h2 agent | h2 `SILENT` after about 20 s, alert |
| Link failure | h3 link down / up | h3 `SILENT`, recovers after link up |
| Packet loss | 25% loss on h3 | missing telemetry windows on h3 |

Screenshots of these scenarios are in [SCREENSHOTS.md](SCREENSHOTS.md).

## 6. Limitations and Future Work

- Loss **before** the switch is not detected because the agent does not send sequence numbers.
- Mitigation drops the whole flow; an OpenFlow meter (rate limit) would be gentler.
- Single-switch topology and a fixed 10 Mbit/s capacity assumption.
- Telemetry flood is detected but not blocked.

## 7. Repository Layout

```
.
├── agent.py  server.py  part_b_topology.py  simple_switch.py
├── monitor_ctrl.py  dashboard.py  run_all.py  summarize_alerts.py
├── README.md
├── CONTRIBUTIONS.md
├── SCREENSHOTS.md
├── docs/
│   ├── ARCHITECTURE.md
│   ├── architecture.png
│   └── screenshots/
├── evidence/        outputs of the automated demo
└── results/         alert logs of the comparison runs
```
