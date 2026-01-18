# Developer and Server Manager Guidance

As the server manager, your primary focus is on maintaining the cluster's stability, security, and performance. This section provides detailed guidance on secure access, node management, and the control of specialized resources like GPUs.

---

## Secure Cluster Access via AWS Systems Manager (SSM)

The traditional method of using a Bastion Host and SSH keys is replaced by **AWS Systems Manager Session Manager**. This provides a more secure, auditable, and keyless method for accessing the Head Node and Compute Nodes.

### Key Benefits of SSM Session Manager

| Benefit | Description |
| :--- | :--- |
| **No SSH Keys** | Access is granted via IAM policies, eliminating the need to manage and rotate SSH key pairs |
| **Auditing** | All session activity is logged to AWS CloudTrail and can be streamed to S3 or CloudWatch Logs |
| **Network Security** | No inbound SSH ports (22) need to be opened in the Security Group |
| **Centralized Control** | IAM policies provide granular, role-based access control |

### Access Procedure

#### Prerequisites

1. **IAM Policy:** Ensure your IAM user has the `ssm:StartSession` permission
2. **Instance Role:** EC2 instances must have an IAM role that grants SSM communication permissions
3. **SSM Agent:** AWS ParallelCluster AMIs include the SSM Agent pre-installed

#### Connect via AWS Console

1. Navigate to **AWS Console → Systems Manager → Session Manager**
2. Click **Start session**
3. Select the Head Node or Compute Node instance ID
4. Click **Start session** to open a shell in your browser

#### Connect via AWS CLI

```bash
# List available instances
aws ssm describe-instance-information --query 'InstanceInformationList[*].[InstanceId,ComputerName,PingStatus]'

# Start session to Head Node
aws ssm start-session --target i-0123456789abcdef0

# Start session with port forwarding (for Jupyter, etc.)
aws ssm start-session \
    --target i-0123456789abcdef0 \
    --document-name AWS-StartPortForwardingSession \
    --parameters '{"portNumber":["8888"],"localPortNumber":["8888"]}'
```

---

## Node Roles and Responsibilities

The cluster is composed of two distinct types of nodes, each with a specific function in the HPC workflow.

| Node Type | Role | Key Components | State | Access |
| :--- | :--- | :--- | :--- | :--- |
| **Head Node** | Control Plane | SLURM Controller, Nextflow Engine, Lmod/Spack, Wazuh Agent, Slurm Exporter, SSM Agent | Static (Always On) | Primary access point for users and managers (via SSM) |
| **Compute Nodes** | Data Plane | Nextflow Processes, Wazuh Agent, Node Exporter, DCGM Exporter (GPU nodes), SSM Agent | Dynamic (Autoscaled) | Launched and terminated by SLURM/ParallelCluster |

### Important Notes

- The **Head Node** is critical and persistent—treat it as a single point of control
- **Compute Nodes** are ephemeral—any direct changes will be lost upon termination
- Configuration changes should be made through ParallelCluster bootstrap scripts or custom AMIs

---

## Controlling CPU and GPU Resources via SLURM

Resource allocation is managed entirely through the SLURM scheduler, which is configured by AWS ParallelCluster to manage the autoscaling groups.

### CPU/GPU Partitioning

| Partition | Purpose | Instance Types | SLURM Option |
| :--- | :--- | :--- | :--- |
| **CPU** | Multi-threaded, non-accelerated tasks | c6a/c7i, Graviton | `-p cpu` |
| **GPU** | Accelerated tasks (DeepVariant) | g5/g6 with NVIDIA GPUs | `-p gpu --gres=gpu:1` |

### SLURM Commands Reference

```bash
# View cluster status
sinfo

# View job queue
squeue

# Submit a CPU job
sbatch --partition=cpu --cpus-per-task=16 --mem=32G script.sh

# Submit a GPU job
sbatch --partition=gpu --cpus-per-task=8 --mem=32G --gres=gpu:1 script.sh

# Check job efficiency
sacct -j <job_id> --format=JobID,JobName,Elapsed,TotalCPU,AllocCPUS,State

# Cancel a job
scancel <job_id>
```

### Developer Guidance for Resource Requests

Nextflow processes must correctly specify resource requirements to ensure jobs are routed to the correct partition.

| Resource | SLURM Directive | Nextflow Directive | Purpose |
| :--- | :--- | :--- | :--- |
| **CPU Cores** | `--cpus-per-task=N` | `cpus N` | Number of CPU cores |
| **Memory** | `--mem=M` | `memory M.GB` | Amount of memory (e.g., 16GB) |
| **GPU** | `--gres=gpu:N` | `clusterOptions '--gres=gpu:N'` | Number of GPUs required |
| **Partition** | `-p partition` | `queue 'partition'` | Target compute partition |

---

## CUDA and GPU Software Management

The most robust method for managing CUDA and other GPU-dependent software is through **Spack** and **Lmod**, ensuring version isolation and reproducibility.

### GPU Software Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                      Application Layer                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ DeepVariant │  │  GATK-GPU   │  │   Custom    │             │
│  │   (Spack)   │  │   (Spack)   │  │   Tools     │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         └────────────────┴────────────────┘                     │
│                          │                                      │
│  ┌───────────────────────▼───────────────────────┐             │
│  │           CUDA Toolkit (Spack/Lmod)           │             │
│  │         cuda@11.8, cuda@12.2, etc.            │             │
│  └───────────────────────┬───────────────────────┘             │
│                          │                                      │
└──────────────────────────┼──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    NVIDIA Driver (AMI)                          │
│              Installed in GPU Compute Node AMI                  │
└─────────────────────────────────────────────────────────────────┘
```

### Setup Steps

#### 1. Base Installation (AMI)

The GPU Compute Node AMI (specified in ParallelCluster config) must include:
- NVIDIA drivers (matching GPU hardware)
- CUDA toolkit runtime

AWS provides optimized Deep Learning AMIs suitable for this purpose.

#### 2. Spack for Applications

Use Spack to install GPU-accelerated applications with specified CUDA version:

```bash
# Install DeepVariant with CUDA 12.2
spack install deepvariant ^cuda@12.2

# Install with specific compiler
spack install deepvariant %gcc@11.3 ^cuda@12.2

# View installed packages
spack find --deps deepvariant
```

#### 3. Lmod for Environment

Load the specific software environment before running jobs:

```bash
# Load DeepVariant with CUDA support
module load deepvariant/1.6-cuda-12.2

# Verify CUDA is accessible
nvcc --version
nvidia-smi

# Check loaded modules
module list
```

### Example Job Script

```bash
#!/bin/bash
#SBATCH --job-name=deepvariant
#SBATCH --partition=gpu
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --gres=gpu:1
#SBATCH --time=02:00:00

# Load required modules
module load deepvariant/1.6-cuda-12.2

# Verify GPU access
nvidia-smi

# Run DeepVariant
run_deepvariant \
    --model_type=WGS \
    --ref=/fsx/reference/GRCh38.fa \
    --reads=/fsx/data/sample.bam \
    --output_vcf=/fsx/results/sample.vcf.gz \
    --num_shards=8
```

---

## Best Practices Summary

| Area | Best Practice |
| :--- | :--- |
| **Access** | Always use SSM Session Manager; never expose SSH |
| **Jobs** | Specify accurate resource requirements to avoid waste |
| **GPU** | Only request GPUs when actually needed |
| **Software** | Use Spack/Lmod for reproducible environments |
| **Monitoring** | Check Grafana dashboards for performance insights |
| **Costs** | Review SLURM accounting for optimization opportunities |
