# Security and Observability

Security and observability are implemented with a decoupled, containerized approach to ensure minimal impact on HPC performance and maximum operational resilience.

---

## Security Architecture

Security is implemented at the host and network level, focusing on detection, auditing, and vulnerability management.

### Host-Based Security (Wazuh)

**Wazuh** is deployed as a host-based security monitoring platform across the entire cluster.

#### Deployment Model

- **Wazuh Manager:** Deployed as a container on **ECS Fargate**. This centralizes logging and security event analysis outside of the HPC cluster, ensuring the Head Node is not burdened with management tasks.
- **Wazuh Agents:** Installed on the Head Node and all Compute Nodes. They report security events and logs back to the Wazuh Manager in the ECS Fargate environment.

#### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    ECS Fargate Cluster                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              Wazuh SIEM Manager                            │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │ │
│  │  │ Log Analysis│  │   Alerts    │  │  Dashboard  │       │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ Security Events / Logs
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   Head Node   │    │  CPU Compute  │    │  GPU Compute  │
│  Wazuh Agent  │    │  Wazuh Agent  │    │  Wazuh Agent  │
└───────────────┘    └───────────────┘    └───────────────┘
```

#### Key Controls

| Control | Description | Monitored Paths |
| :--- | :--- | :--- |
| **File Integrity Monitoring (FIM)** | Monitors critical system files for unauthorized changes | `/etc`, `/opt/apps`, `/home` |
| **Intrusion Detection** | Detects common attack patterns and brute-force attempts | SSH logs, system logs |
| **Audit Logging** | Centralizes and analyzes OS and application logs | All system logs |
| **User Activity Auditing** | Tracks user commands and actions | Shell history, sudo logs |

### Vulnerability Management (Trivy)

**Trivy** is used for scheduled vulnerability scanning to maintain the security posture of the software stack.

| Attribute | Configuration |
| :--- | :--- |
| **Scope** | Head Node OS, installed software in `/opt/apps` |
| **Frequency** | Weekly or monthly, on-demand after updates |
| **Output** | JSON/HTML reports, integration with CI/CD |
| **Rationale** | Avoids runtime overhead on compute nodes |

---

## Monitoring and Observability

A robust observability stack is critical for managing cluster health, performance, and cost. The design utilizes **Prometheus** and **Grafana**, both deployed on **ECS Fargate**.

### Observability Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    ECS Fargate Cluster                          │
│  ┌─────────────────┐              ┌─────────────────┐          │
│  │   Prometheus    │◄────────────►│    Grafana      │          │
│  │  (Time-Series)  │              │  (Dashboards)   │          │
│  └────────┬────────┘              └────────┬────────┘          │
│           │                                │                    │
└───────────┼────────────────────────────────┼────────────────────┘
            │ Scrape                         │ HTTPS
            ▼                                ▼
┌───────────────────────────────────────────────────────────────┐
│                     HPC Cluster                                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐ │
│  │   Head Node     │  │   CPU Nodes     │  │   GPU Nodes    │ │
│  │ • Node Exporter │  │ • Node Exporter │  │ • Node Exporter│ │
│  │ • Slurm Exporter│  │                 │  │ • DCGM Exporter│ │
│  └─────────────────┘  └─────────────────┘  └────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

### Prometheus Exporters

The following exporters are deployed to gather comprehensive metrics:

| Exporter | Deployment | Key Metrics | Focus Area |
| :--- | :--- | :--- | :--- |
| **Node Exporter** | Head + Compute | `node_cpu_seconds_total`, `node_memory_MemAvailable_bytes`, `node_disk_read_bytes_total` | Infrastructure Health |
| **Slurm Exporter** | Head Node | `slurm_job_count`, `slurm_partition_up`, `slurm_job_cpu_efficiency` | Scheduler & Workload |
| **DCGM Exporter** | GPU Nodes | `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_MEM_COPY_UTIL`, `DCGM_FI_DEV_POWER_USAGE` | GPU Performance |
| **Nextflow Reports** | Head Node | Process-level metrics (CPU time, memory, I/O) | Pipeline Efficiency |

### Grafana Dashboards

Grafana provides visualization through pre-built and custom dashboards. Access is secured via **Route53** and **ACM** over HTTPS.

| Dashboard | Exporter | Grafana ID | Key Visualizations |
| :--- | :--- | :--- | :--- |
| **Cluster Health Overview** | Node Exporter | 1860 | CPU/Memory utilization, network, disk |
| **SLURM Workload Analysis** | Slurm Exporter | 4323 | Queue length, job states, node status |
| **GPU Performance Monitor** | DCGM Exporter | 12239 | GPU utilization, memory, temperature |
| **Nextflow Pipeline Metrics** | Custom | N/A | Job efficiency, runtime per stage |

### Example Dashboard Queries

```promql
# CPU utilization across all compute nodes
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# SLURM pending jobs
slurm_job_count{state="pending"}

# GPU utilization
DCGM_FI_DEV_GPU_UTIL{gpu="0"}

# Memory usage percentage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

---

## Alerting Strategy

Alerts are configured through Prometheus Alertmanager with actionable notifications.

### Alert Channels

| Channel | Use Case |
| :--- | :--- |
| **Email** | Critical alerts, daily summaries |
| **Slack** | Real-time notifications (optional) |
| **AWS SNS** | Integration with AWS services |

### Example Alerts

| Alert | Condition | Severity |
| :--- | :--- | :--- |
| **Head Node Overload** | CPU > 90% for 5 min | Critical |
| **GPU Underutilization** | GPU < 10% during job | Warning |
| **Job Queue Saturation** | Pending > 100 jobs | Warning |
| **Node Failure** | Node down > 2 min | Critical |
| **Disk Space Low** | Available < 10% | Warning |

### Alertmanager Configuration Example

```yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
  - name: 'default'
    email_configs:
      - to: 'hpc-admin@example.com'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/xxx'
        channel: '#hpc-alerts'
```

---

## Security Best Practices

1. **Least Privilege:** IAM roles grant minimum required permissions
2. **Network Isolation:** Private subnets for compute, public only for ALB
3. **Encryption:** TLS for all communications, EBS encryption at rest
4. **Audit Trail:** CloudTrail logging for all API calls
5. **Regular Scanning:** Scheduled vulnerability assessments
6. **Incident Response:** Documented procedures for security events
