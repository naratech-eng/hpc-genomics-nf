# System Design Flow: Nextflow Pipeline Execution

This section details the complete workflow from user submission to final results archival, highlighting the interaction between the user, the Nextflow engine, the SLURM scheduler, and the multi-tiered storage system.

---

## Pipeline Execution Sequence

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        PIPELINE EXECUTION FLOW                               │
└──────────────────────────────────────────────────────────────────────────────┘

    User                Head Node              SLURM           Compute Nodes
     │                     │                     │                   │
     │  1. SSH via SSM     │                     │                   │
     │────────────────────►│                     │                   │
     │                     │                     │                   │
     │  2. nextflow run    │                     │                   │
     │────────────────────►│                     │                   │
     │                     │                     │                   │
     │                     │  3. Stage data      │                   │
     │                     │  from S3 to FSx     │                   │
     │                     │─────────────────────┼──────────────────►│
     │                     │                     │                   │
     │                     │  4. Submit CPU job  │                   │
     │                     │────────────────────►│                   │
     │                     │                     │  5. Launch node   │
     │                     │                     │──────────────────►│
     │                     │                     │                   │
     │                     │                     │  6. Execute task  │
     │                     │                     │◄──────────────────│
     │                     │                     │                   │
     │                     │  7. Submit GPU job  │                   │
     │                     │────────────────────►│                   │
     │                     │                     │  8. Launch GPU    │
     │                     │                     │──────────────────►│
     │                     │                     │                   │
     │                     │                     │  9. DeepVariant   │
     │                     │                     │◄──────────────────│
     │                     │                     │                   │
     │                     │  10. Archive to S3  │                   │
     │                     │─────────────────────┼──────────────────►│
     │                     │                     │                   │
     │  11. Notification   │                     │                   │
     │◄────────────────────│                     │                   │
     │                     │                     │                   │
```

---

## Step-by-Step Guidance

### Step 1: User Submission

The user connects to the Head Node via AWS Systems Manager Session Manager and executes the Nextflow pipeline command.

```bash
# Connect via SSM (AWS CLI)
aws ssm start-session --target i-0123456789abcdef0

# Execute the pipeline
nextflow run main.nf -profile slurm --input samples.csv
```

### Step 2: Data Staging

The Nextflow script or a pre-job hook stages the necessary raw data from S3 to the high-speed FSx for Lustre file system.

```bash
# Data staging from S3 to FSx
aws s3 sync s3://genomics-bucket/raw-data/ /fsx/raw-data/
```

### Step 3: Job Submission Loop

Nextflow iterates through the pipeline stages:

1. **Submits a SLURM job request** for each process (e.g., alignment, variant calling)
2. **SLURM checks resources** and provisions a suitable Compute Node if none are available
3. **Compute Node executes** the task, reading/writing to FSx for Lustre
4. **Monitoring agents report** metrics to the ECS Fargate management plane
5. **Compute Node signals** job completion to SLURM

### Step 4: Finalization

Once the pipeline completes, the Head Node orchestrates the transfer of final results from FSx for Lustre back to the long-term S3 archive.

```bash
# Archive results to S3
aws s3 sync /fsx/results/ s3://genomics-bucket/results/
```

### Step 5: Notification

The user is notified of the pipeline completion via configured channels (email, Slack, etc.).

---

## Pipeline Stages Detail

| Stage | Partition | Resources | Duration | Output |
| :--- | :--- | :--- | :--- | :--- |
| **Data Download** | CPU | 4 cores, 8GB | ~30 min | Raw FASTQ files |
| **Quality Control** | CPU | 4 cores, 8GB | ~15 min | QC reports |
| **Alignment** | CPU | 16 cores, 32GB | ~2 hrs | BAM files |
| **Variant Calling** | GPU | 8 cores, 32GB, 1 GPU | ~1 hr | VCF files |
| **Post-processing** | CPU | 4 cores, 16GB | ~30 min | Filtered VCF |

---

## Resource Request Examples

### Nextflow Process Definitions

```groovy
// CPU-intensive alignment process
process ALIGNMENT {
    cpus 16
    memory '32 GB'
    time '4h'
    queue 'cpu'
    
    input:
    tuple val(sample_id), path(reads)
    
    output:
    tuple val(sample_id), path("${sample_id}.bam")
    
    script:
    """
    module load bwa/0.7.17
    module load samtools/1.17
    
    bwa mem -t ${task.cpus} \
        ${reference} \
        ${reads[0]} ${reads[1]} | \
    samtools sort -@ ${task.cpus} -o ${sample_id}.bam
    """
}

// GPU-accelerated variant calling
process VARIANT_CALLING {
    cpus 8
    memory '32 GB'
    time '2h'
    queue 'gpu'
    clusterOptions '--gres=gpu:1'
    
    input:
    tuple val(sample_id), path(bam)
    
    output:
    tuple val(sample_id), path("${sample_id}.vcf.gz")
    
    script:
    """
    module load deepvariant/1.6-cuda-12.2
    
    run_deepvariant \
        --model_type=WGS \
        --ref=${reference} \
        --reads=${bam} \
        --output_vcf=${sample_id}.vcf.gz \
        --num_shards=${task.cpus}
    """
}
```

---

## Observability During Execution

Throughout pipeline execution, the following metrics are continuously collected:

| Metric Source | Data Collected | Destination |
| :--- | :--- | :--- |
| **Node Exporter** | CPU, memory, disk I/O | Prometheus |
| **SLURM Exporter** | Job states, queue length | Prometheus |
| **DCGM Exporter** | GPU utilization, memory | Prometheus |
| **Wazuh Agent** | Security events, file integrity | Wazuh Manager |
| **Nextflow** | Process metrics, timing | Reports (HTML/CSV) |

All metrics are visualized in real-time through Grafana dashboards accessible via the secure HTTPS endpoint.
