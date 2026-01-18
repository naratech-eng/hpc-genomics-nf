# Post-Deployment Validation Checklist

This checklist is designed for the Server Manager to validate the successful deployment and operational readiness of the AWS ParallelCluster HPC environment and its core components. All checks should be performed via **AWS Systems Manager Session Manager** on the Head Node, and where indicated, on a newly launched Compute Node.

---

## Pre-Validation Requirements

Before beginning validation:

- [ ] AWS Console access with appropriate permissions
- [ ] AWS CLI configured with correct credentials
- [ ] Terraform apply completed successfully
- [ ] ParallelCluster status shows `CREATE_COMPLETE`

---

## Infrastructure and Access Checks (Head Node)

| Check | Procedure | Expected Result | Status |
| :--- | :--- | :--- | :--- |
| **SSM Access** | Connect via AWS Console → Session Manager | Successful shell session without SSH keys | ☐ |
| **Storage Mounts** | Run `df -h` | `/shared` (EFS) and `/fsx` (FSx) mounted | ☐ |
| **Nextflow Installation** | Run `nextflow -version` | Nextflow version displayed | ☐ |
| **Spack/Lmod Setup** | Run `module avail` | List of installed modules displayed | ☐ |
| **Prometheus Target** | Check Prometheus UI or Grafana | Head Node listed as `up` target | ☐ |

### Validation Commands

```bash
# Test SSM access (run from local machine)
aws ssm start-session --target <head-node-instance-id>

# Verify storage mounts
df -h | grep -E '(fsx|shared|efs)'

# Expected output:
# /dev/nvme1n1     1.2T  100G  1.1T   9% /fsx
# fs-xxx.efs...    8.0E  100G  8.0E   1% /shared

# Verify Nextflow
nextflow -version

# Verify Spack modules
module avail 2>&1 | head -20
```

---

## SLURM and Autoscaling Checks (Head Node)

| Check | Procedure | Expected Result | Status |
| :--- | :--- | :--- | :--- |
| **SLURM Status** | Run `sinfo` | All partitions (`cpu`, `gpu`) listed | ☐ |
| **CPU Autoscaling** | Submit test CPU job | Node launches, job runs, node terminates | ☐ |
| **GPU Autoscaling** | Submit test GPU job | GPU node launches, `nvidia-smi` succeeds | ☐ |
| **Slurm Exporter** | Check Grafana dashboard | `slurm_job_count` metric visible | ☐ |

### Validation Commands

```bash
# Check SLURM partitions
sinfo
# Expected:
# PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
# cpu*         up   infinite      0  idle~ cpu-dy-c6a-[1-10]
# gpu          up   infinite      0  idle~ gpu-dy-g5-[1-4]

# Test CPU autoscaling
sbatch --wrap="hostname && sleep 30" -p cpu -t 5:00
squeue  # Monitor job state

# Test GPU autoscaling
sbatch --wrap="nvidia-smi" --gres=gpu:1 -p gpu -t 5:00
squeue  # Monitor job state

# Check job output after completion
cat slurm-<job_id>.out
```

---

## Compute Node Component Checks

These checks require submitting a job that runs validation scripts on the Compute Node.

| Check | Procedure | Expected Result | Status |
| :--- | :--- | :--- | :--- |
| **Wazuh Agent** | Check agent status from Head Node | Compute Node agent `Active` | ☐ |
| **GPU Driver/CUDA** | Run `nvidia-smi` on GPU node | Driver and GPU info displayed | ☐ |
| **Lmod on Compute** | Load module on Compute Node | Module loads, binary path resolved | ☐ |
| **DCGM Exporter** | Check GPU dashboard | GPU metrics flowing | ☐ |
| **Node Exporter** | Check Cluster Health dashboard | CPU/Memory metrics flowing | ☐ |

### Validation Commands

```bash
# Check Wazuh agent connections from Head Node
sudo /var/ossec/bin/agent_control -l
# Look for compute node IPs with "Active" status

# Comprehensive GPU node test
cat << 'EOF' > /fsx/validate-gpu.sh
#!/bin/bash
echo "=== GPU Validation Script ==="
echo "Hostname: $(hostname)"
echo ""
echo "=== NVIDIA Driver ==="
nvidia-smi
echo ""
echo "=== CUDA Version ==="
nvcc --version 2>/dev/null || echo "nvcc not in PATH"
echo ""
echo "=== Module System ==="
module avail 2>&1 | head -10
echo ""
echo "=== Storage Mounts ==="
df -h | grep -E '(fsx|shared|efs)'
echo ""
echo "=== Wazuh Agent ==="
systemctl status wazuh-agent | head -5
echo ""
echo "=== Node Exporter ==="
curl -s localhost:9100/metrics | head -5
echo "=== Validation Complete ==="
EOF

chmod +x /fsx/validate-gpu.sh

# Submit validation job
sbatch --wrap="/fsx/validate-gpu.sh" --gres=gpu:1 -p gpu -o /fsx/gpu-validation.out
```

---

## Management Plane Checks (ECS Fargate)

| Check | Procedure | Expected Result | Status |
| :--- | :--- | :--- | :--- |
| **ECS Services** | Check ECS console | All services `RUNNING` | ☐ |
| **Wazuh Manager** | Access Wazuh dashboard | Dashboard accessible, agents visible | ☐ |
| **Prometheus** | Access Prometheus UI | Targets healthy, metrics collecting | ☐ |
| **Grafana** | Access Grafana via HTTPS | Dashboards loading, data visible | ☐ |
| **TLS Certificate** | Check browser | Valid certificate (ACM) | ☐ |

### Validation Commands

```bash
# Check ECS service status
aws ecs list-services --cluster hpc-management
aws ecs describe-services --cluster hpc-management --services wazuh prometheus grafana

# Get Grafana URL from Terraform output
terraform output grafana_url

# Test Grafana access
curl -I https://grafana.your-domain.com
# Expected: HTTP/2 200 or 302 (redirect to login)
```

---

## End-to-End Pipeline Test

| Check | Procedure | Expected Result | Status |
| :--- | :--- | :--- | :--- |
| **Sample Pipeline** | Run test Nextflow pipeline | Pipeline completes successfully | ☐ |
| **Data Staging** | Verify S3 to FSx transfer | Data visible in `/fsx` | ☐ |
| **Results Archive** | Verify FSx to S3 transfer | Results in S3 bucket | ☐ |
| **Metrics Correlation** | Check Grafana during run | Job metrics visible | ☐ |

### Sample Test Pipeline

```bash
# Create a simple test pipeline
cat << 'EOF' > /shared/test-pipeline.nf
#!/usr/bin/env nextflow
nextflow.enable.dsl=2

process CPU_TEST {
    cpus 2
    memory '4 GB'
    queue 'cpu'
    
    output:
    path 'cpu_result.txt'
    
    script:
    """
    echo "CPU test on \$(hostname)" > cpu_result.txt
    echo "Date: \$(date)" >> cpu_result.txt
    sleep 30
    """
}

process GPU_TEST {
    cpus 2
    memory '8 GB'
    queue 'gpu'
    clusterOptions '--gres=gpu:1'
    
    output:
    path 'gpu_result.txt'
    
    script:
    """
    echo "GPU test on \$(hostname)" > gpu_result.txt
    nvidia-smi >> gpu_result.txt
    """
}

workflow {
    CPU_TEST()
    GPU_TEST()
}
EOF

# Run the test pipeline
cd /shared
nextflow run test-pipeline.nf -profile slurm

# Check results
cat work/*/cpu_result.txt
cat work/*/gpu_result.txt
```

---

## Validation Summary

### Sign-Off

| Component | Validated By | Date | Notes |
| :--- | :--- | :--- | :--- |
| Infrastructure | | | |
| SLURM | | | |
| Compute Nodes | | | |
| Management Plane | | | |
| End-to-End | | | |

### Issues Found

| Issue | Severity | Resolution | Status |
| :--- | :--- | :--- | :--- |
| | | | |

---

## Post-Validation Actions

After successful validation:

- [ ] Document any configuration changes
- [ ] Update runbook with environment-specific details
- [ ] Configure alerting thresholds based on baseline metrics
- [ ] Schedule regular validation runs (monthly recommended)
- [ ] Archive validation logs for compliance
