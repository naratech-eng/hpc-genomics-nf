# Technology Stack & Design Decisions

This section details the technology choices and rationale behind the compute, storage, workflow, and software management components of the HPC environment.

---

## Compute and Scheduling

The design leverages SLURM's partitioning feature to match workloads to optimized hardware. This ensures that the right compute resources are used for the right task, maximizing performance and minimizing cost.

| Partition | Workload Type | Recommended Instance Family | Rationale |
| :--- | :--- | :--- | :--- |
| **CPU** | Alignment (BWA-MEM), Quality Control (FastQC), Pre-processing | **c6a/c7i** (Compute-Optimized) or **Graviton3** (Cost-Optimized) | High core count and clock speed for multi-threaded tasks. Graviton3 offers superior price-performance [5]. **Spot Instances** are recommended for cost savings [2]. |
| **GPU** | Variant Calling (DeepVariant), GPU-accelerated tasks | **g5/g6** (GPU-Optimized) | Provides NVIDIA GPUs (e.g., A10G, H100) necessary for accelerated deep learning-based variant callers like DeepVariant [4]. |

### Instance Selection Guidelines

- **c6a/c7i:** Best for I/O-bound and compute-intensive CPU tasks
- **Graviton3 (c7g):** 40% better price-performance for compatible workloads
- **g5:** Cost-effective GPU instances with A10G GPUs
- **g6:** Latest generation with improved performance

---

## Storage Architecture

A multi-tiered storage strategy is employed to balance performance, cost, and persistence. This approach ensures high-speed access for active processing while maintaining cost-effective, durable storage for raw and final data.

| Storage Service | Purpose | Performance Profile | Data Lifecycle Stage |
| :--- | :--- | :--- | :--- |
| **Amazon S3** | Source Data & Archive | High durability, low cost, object storage. | Raw data ingestion, final results archive. |
| **FSx for Lustre** | Scratch/Working Data | High IOPS, low latency, parallel file system. | Intermediate files, alignment indices, active processing. **Data is staged from S3.** |
| **Amazon EFS** | Persistent Home/Config | Shared, concurrent access, moderate performance. | User home directories, Nextflow scripts, Spack environment modules. |

### Storage Workflow

```
┌─────────────┐     Stage Data     ┌─────────────────┐     Process     ┌─────────────┐
│  Amazon S3  │ ─────────────────► │ FSx for Lustre  │ ─────────────► │   Results   │
│ (Raw Data)  │                    │   (Scratch)     │                │  (Archive)  │
└─────────────┘                    └─────────────────┘                └─────────────┘
                                          │
                                          │ Read/Write
                                          ▼
                                   ┌─────────────────┐
                                   │  Compute Nodes  │
                                   └─────────────────┘
```

---

## Workflow Orchestration

**Nextflow** is selected as the workflow engine due to its native support for complex pipeline logic, checkpointing, and seamless integration with the SLURM scheduler [3].

### Execution Model

- Nextflow submits each process as a separate job to the SLURM scheduler
- Resource requirements are declared per stage
- Automatic retries and checkpointing handle failures gracefully

### Resource Awareness

Each Nextflow process definition explicitly requests resources (CPUs, memory, time, and GRES for GPUs), allowing SLURM to intelligently place the job on the correct compute partition.

```groovy
process VARIANT_CALLING {
    cpus 8
    memory '32 GB'
    time '4h'
    clusterOptions '--gres=gpu:1 -p gpu'
    
    script:
    """
    deepvariant --input ${bam} --output ${vcf}
    """
}
```

### Key Benefits

- **Scalable:** Handles thousands of concurrent jobs
- **Fault-tolerant:** Automatic retry and resume capabilities
- **Portable:** Runs on cloud and on-premises HPC systems
- **Reproducible:** Version-controlled pipeline definitions

---

## Software Management

**Spack** and **Lmod** are the chosen tools for software management, mirroring best practices in traditional HPC environments.

### Spack

Used to build and install all bioinformatics tools from source, ensuring version isolation and dependency tracking. This allows for optimization of binaries for the specific EC2 instance architecture (e.g., Graviton or Intel).

**Installed Tools:**
- `bwa` - Alignment
- `samtools` - BAM/SAM manipulation
- `fastqc` - Quality control
- `bcftools` - VCF manipulation
- `deepvariant` - Variant calling
- `cuda` - GPU support

### Lmod

Provides a dynamic module system, allowing users to easily load and unload specific versions of software, ensuring a clean and reproducible environment for each Nextflow process.

```bash
# Load specific software versions
module load bwa/0.7.17
module load samtools/1.17
module load deepvariant/1.6-cuda-12.2

# Check loaded modules
module list
```

### Benefits

- **Version Isolation:** Multiple versions coexist without conflicts
- **Dependency Tracking:** Automatic resolution of software dependencies
- **Reproducibility:** Exact software environments can be recreated
- **HPC Compatibility:** Matches traditional HPC center practices
