# Project Architecture and Design

## 1. Architecture Diagram

 h1 10.0.0.1 --- s1-eth1 \
 h2 10.0.0.2 --- s1-eth2  \
 h3 10.0.0.3 --- s1-eth3   s1 (OpenFlow 1.3) ---- os-ken controller (127.0.0.1:6653)
 server 10.0.0.100 s1-eth4 /          |
   UDP port 5000 <-- telemetry        +-- /tmp/sdn_stats.json --> web dashboard :8080
```

## 2. Network Topology

- 1 OpenFlow 1.3 switch (`s1`)
- 3 hosts (`h1`, `h2`, `h3`) and 1 monitoring server (`10.0.0.100`)
- 10 Mbit/s links
- Remote controller (os-ken) at `127.0.0.1:6653`
- Telemetry sent over UDP to port 5000

## 3. Components

| Component | File | Role |
|---|---|---|
| Telemetry agent | `agent.py` | runs on h1-h3, sends a UDP message every 5 s |
| Monitoring server | `server.py` | receives UDP on `10.0.0.100:5000`, logs to `/tmp/server.log` |
| Topology | `part_b_topology.py` | Mininet: 1 switch, 3 hosts, 1 server, 10 Mbit/s links, remote controller |
| Basic controller | `simple_switch.py` | learning switch (used for the initial SDN test) |
| Monitoring controller | `monitor_ctrl.py` | stats polling, utilization, alerts, mitigation, correlation |
| Dashboard | `dashboard.py` | web UI (standard library only) with scenario buttons |
| Launcher | `run_all.py` | starts controller, network, agents, server and dashboard |
| Log analysis | `summarize_alerts.py` | compares alert logs of different runs |

## 4. How the Pieces Work Together

1. The switch sends unknown packets to the controller, which installs per-flow rules
   (IPv4 source/destination and UDP/TCP destination port). Telemetry (port 5000) is therefore a
   separate, countable flow.
2. Every 5 s the controller reads **port statistics** (utilization = bytes x 8 / time / link
   capacity) and **flow statistics** (per-host telemetry packet counts).
3. It reads the server log and compares *packets the switch forwarded* with *packets the server
   logged* for each host.
4. Alerts are raised according to the table below.
5. On CRITICAL the controller installs a high-priority **drop rule** for the heaviest
   non-telemetry flow for 20 s (disable with `--mitigate 0` to compare).

## 5. Data Flow

```
Agents (h1-h3) --UDP:5000--> Switch s1 --> Monitoring server --> /tmp/server.log
                                |                                      |
                                v                                      v
                        os-ken controller <---------- correlation ------+
                                |
                                +--> alerts + mitigation (drop rule on s1)
                                |
                                +--> /tmp/sdn_stats.json --> Web dashboard :8080
```

## 6. Alert Rules

| Alert | Condition |
|---|---|
| Link utilization WARNING | utilization >= 60% |
| Link utilization CRITICAL | utilization >= 85% |
| Host silent | no telemetry for more than 20 s |
| Telemetry flood | more than 20 packets/s |
| Loss after the switch | more than 30% of forwarded packets missing at the server |

## 7. Mitigation

When a link becomes CRITICAL, the controller finds the heaviest non-telemetry flow and installs a
high-priority drop rule for it for 20 s. Telemetry traffic is not dropped, so monitoring continues.

## 8. Protocols and Technologies

| Item | Use |
|---|---|
| UDP sockets | telemetry from agents to server |
| OpenFlow 1.3 | controller-switch communication, flow rules, statistics |
| Mininet / Open vSwitch | network emulation |
| os-ken 2.8.1 | SDN controller framework |
| iperf | traffic generation for congestion tests |
| Python (standard library) | dashboard and tools |

## 9. Limitations

- Loss **before** the switch is not detected because the agent does not send sequence numbers.
- Mitigation drops the whole flow; an OpenFlow meter (rate limit) would be gentler.
- Single-switch topology and a fixed 10 Mbit/s capacity assumption.
- Telemetry flood is detected but not blocked.
