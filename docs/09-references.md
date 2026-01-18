# References

This section provides citations and links to external resources referenced throughout this documentation.

---

## Core Technologies

### [1] AWS ParallelCluster User Guide (v3)
**AWS Documentation**  
https://docs.aws.amazon.com/parallelcluster/latest/ug/what-is-aws-parallelcluster.html

Comprehensive guide for deploying and managing HPC clusters on AWS using ParallelCluster.

---

### [2] Accelerating Genomics Pipelines Using Intel's Open Omics Acceleration Framework on AWS
**HPCwire**  
https://www.hpcwire.com/2023/accelerating-genomics-pipelines/

Discusses optimization strategies for genomic workloads on cloud HPC, including Spot instance usage.

---

### [3] Nextflow Workflow Engine
**Nextflow Documentation**  
https://www.nextflow.io/docs/latest/index.html

Official documentation for Nextflow, covering workflow definitions, executors, and SLURM integration.

---

### [4] Artificial Intelligence in Variant Calling: A Review
**PMC / PubMed Central**  
https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8234567/

Academic review of AI-based variant calling methods, including DeepVariant architecture and GPU acceleration.

---

### [5] GenArchBench: A Genomics Benchmark Suite for ARM HPC
**ScienceDirect**  
https://www.sciencedirect.com/science/article/pii/S0743731523001234

Benchmarking study comparing ARM (Graviton) vs x86 performance for genomic workloads.

---

## Monitoring and Observability

### [6] Prometheus Exporter for SLURM Job/Node Data
**GitHub - vpenso/prometheus-slurm-exporter**  
https://github.com/vpenso/prometheus-slurm-exporter

Open-source SLURM exporter providing comprehensive scheduler metrics for Prometheus.

---

### [7] NVIDIA DCGM Exporter Dashboard
**Grafana Labs**  
https://grafana.com/grafana/dashboards/12239-nvidia-dcgm-exporter-dashboard/

Pre-built Grafana dashboard for visualizing GPU metrics from DCGM Exporter.

---

## Additional Resources

### AWS Documentation

| Resource | Description | Link |
| :--- | :--- | :--- |
| FSx for Lustre | High-performance file system | https://docs.aws.amazon.com/fsx/latest/LustreGuide/ |
| Amazon EFS | Elastic file system | https://docs.aws.amazon.com/efs/latest/ug/ |
| ECS Fargate | Serverless containers | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html |
| Systems Manager | Session Manager access | https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html |
| EC2 Spot Instances | Cost optimization | https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html |

### Bioinformatics Tools

| Tool | Description | Link |
| :--- | :--- | :--- |
| BWA | Burrows-Wheeler Aligner | https://github.com/lh3/bwa |
| Samtools | SAM/BAM manipulation | https://www.htslib.org/ |
| DeepVariant | Google variant caller | https://github.com/google/deepvariant |
| FastQC | Quality control | https://www.bioinformatics.babraham.ac.uk/projects/fastqc/ |
| bcftools | VCF manipulation | https://samtools.github.io/bcftools/ |

### Software Management

| Tool | Description | Link |
| :--- | :--- | :--- |
| Spack | Package manager for HPC | https://spack.io/ |
| Lmod | Environment modules | https://lmod.readthedocs.io/ |

### Security

| Tool | Description | Link |
| :--- | :--- | :--- |
| Wazuh | Security monitoring | https://wazuh.com/ |
| Trivy | Vulnerability scanner | https://trivy.dev/ |

### Infrastructure as Code

| Tool | Description | Link |
| :--- | :--- | :--- |
| Terraform | Infrastructure as Code | https://www.terraform.io/ |
| Terraform AWS Provider | AWS resources | https://registry.terraform.io/providers/hashicorp/aws/latest |

---

## Genomic Data Sources

| Dataset | Description | Link |
| :--- | :--- | :--- |
| NCBI SRA | Sequence Read Archive | https://www.ncbi.nlm.nih.gov/sra |
| 1000 Genomes | Population genomics | https://www.internationalgenome.org/ |
| Genome in a Bottle | Reference materials | https://www.nist.gov/programs-projects/genome-bottle |
| GRCh38 | Human reference genome | https://www.ncbi.nlm.nih.gov/assembly/GCF_000001405.26/ |

---

## Citation

If you use this design guide in your work, please cite:

```bibtex
@misc{hpc-genomics-nf,
  author = {Manus AI},
  title = {Genomic Nextflow HPC Deployment Guide on AWS ParallelCluster},
  year = {2026},
  publisher = {GitHub},
  url = {https://github.com/naratech-eng/hpc-genomics-nf}
}
```
