# Cost Optimization Strategies

The elastic nature of the cloud is leveraged to minimize the total cost of ownership (TCO) for the HPC environment. Monitoring via the decoupled Prometheus/Grafana stack is essential for validating the effectiveness of these strategies.

---

## Cost Optimization Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                    COST OPTIMIZATION STRATEGIES                      │
└──────────────────────────────────────────────────────────────────────┘

┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Spot Instances │  │  Targeted GPU   │  │   Aggressive    │
│   for CPU Jobs  │  │      Usage      │  │   Autoscaling   │
│                 │  │                 │  │                 │
│   Up to 90%     │  │  GPU only for   │  │  Low idle       │
│   savings       │  │  DeepVariant    │  │  timeout        │
└─────────────────┘  └─────────────────┘  └─────────────────┘

┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ FSx Lifecycle   │  │    Graviton     │  │  Resource       │
│   Management    │  │    Adoption     │  │  Right-Sizing   │
│                 │  │                 │  │                 │
│  Delete scratch │  │  40% better     │  │  Match requests │
│  with cluster   │  │  price-perf     │  │  to actual use  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

---

## Strategy Details

### 1. Spot Instances for CPU Workloads

Utilize **EC2 Spot Instances** for all non-critical, fault-tolerant CPU workloads (e.g., alignment, QC). This can provide up to **90% savings** compared to On-Demand pricing [2].

| Aspect | Configuration |
| :--- | :--- |
| **Applicable Workloads** | Alignment (BWA-MEM), Quality Control (FastQC), Pre-processing |
| **Interruption Handling** | Nextflow retry + checkpointing |
| **Fallback** | On-Demand instances when Spot unavailable |
| **Savings** | 60-90% vs On-Demand |

```hcl
# Terraform example for Spot configuration
resource "aws_parallelcluster" "hpc" {
  scheduling {
    scheduler = "slurm"
    queues {
      name = "cpu"
      compute_settings {
        spot_price = 0.50  # Maximum spot price
      }
      networking {
        placement_group {
          enabled = true
        }
      }
    }
  }
}
```

### 2. Targeted GPU Usage

Restrict the use of expensive GPU instances (g5/g6) exclusively to the DeepVariant stage, which requires GPU acceleration. All other stages run on lower-cost CPU instances.

| Stage | Instance Type | Cost Category |
| :--- | :--- | :--- |
| Data Download | CPU (c6a) | Low |
| Quality Control | CPU (c6a) | Low |
| Alignment | CPU (c6a) | Medium |
| **Variant Calling** | **GPU (g5)** | **High** |
| Post-processing | CPU (c6a) | Low |

### 3. Aggressive Autoscaling

Configure AWS ParallelCluster with a low idle timeout. Compute Nodes are terminated quickly after jobs complete, ensuring you only pay for compute time actively used.

```yaml
# ParallelCluster config example
Scheduling:
  Scheduler: slurm
  SlurmSettings:
    ScaledownIdletime: 5  # Minutes before idle nodes are terminated
  SlurmQueues:
    - Name: cpu
      ComputeResources:
        - Name: cpu-spot
          MinCount: 0
          MaxCount: 20
          InstanceType: c6a.4xlarge
```

### 4. FSx for Lustre Lifecycle Management

Implement a lifecycle policy to automatically manage the FSx for Lustre file system:

| Policy | Description | Cost Impact |
| :--- | :--- | :--- |
| **Delete with cluster** | Scratch FSx deleted on `terraform destroy` | Eliminates lingering storage costs |
| **S3 Integration** | Auto-export results to S3, auto-import raw data | Reduces manual data movement |
| **Persistent option** | Use for long-running projects | Fixed monthly cost |

### 5. Graviton Adoption

Prioritize **Graviton3** instances (c7g) for CPU workloads where possible, as they offer a superior price-performance ratio compared to comparable x86 instances [5].

| Metric | x86 (c6a) | Graviton3 (c7g) | Improvement |
| :--- | :--- | :--- | :--- |
| Price/vCPU/hour | $0.0425 | $0.0340 | 20% lower |
| Performance/vCPU | Baseline | +25% | 25% higher |
| **Total Value** | Baseline | **+40%** | Combined benefit |

**Note:** Ensure bioinformatics tools are compiled for ARM64 architecture using Spack.

---

## Cost Visibility and Tracking

### Per-Job Cost Estimation

Use SLURM accounting combined with AWS Cost Explorer to estimate per-job costs:

```bash
# SLURM job cost query
sacct -j <job_id> --format=JobID,Elapsed,AllocCPUS,AllocTRES%30,MaxRSS

# Calculate cost based on instance type and runtime
```

### Resource Request Tuning

Monitor actual vs. requested resources to optimize future job submissions:

| Metric | Source | Action |
| :--- | :--- | :--- |
| CPU Efficiency | `slurm_job_cpu_efficiency` | Reduce CPU requests if <50% |
| Memory Usage | `MaxRSS` from sacct | Right-size memory requests |
| GPU Utilization | DCGM metrics | Investigate if <30% |

### Cost Dashboard

Create a custom Grafana dashboard tracking:

- Daily compute spend by partition
- Cost per pipeline run
- Spot savings percentage
- Idle resource waste

---

## Cost Optimization Checklist

- [ ] Spot Instances enabled for CPU partition
- [ ] GPU partition restricted to variant calling only
- [ ] Autoscaling idle timeout set to ≤10 minutes
- [ ] FSx for Lustre lifecycle policy configured
- [ ] Graviton instances tested for compatible workloads
- [ ] SLURM accounting enabled for cost tracking
- [ ] Grafana cost dashboard deployed
- [ ] Monthly cost review scheduled

---

## Estimated Monthly Costs

| Component | Configuration | Estimated Cost |
| :--- | :--- | :--- |
| Head Node (On-Demand) | c6a.2xlarge, always on | ~$250/month |
| CPU Compute (Spot) | c6a.4xlarge, 100 hours | ~$50/month |
| GPU Compute (On-Demand) | g5.2xlarge, 20 hours | ~$30/month |
| FSx for Lustre | 1.2 TB, scratch | ~$150/month |
| EFS | 100 GB | ~$30/month |
| ECS Fargate | 3 tasks | ~$50/month |
| **Total** | | **~$560/month** |

*Actual costs vary based on usage patterns and AWS pricing changes.*
