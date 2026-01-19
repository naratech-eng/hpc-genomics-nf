# Genomic Workflow Deployment on AWS ParallelCluster Using Nextflow and SLURM

## Overview

This documentation provides a comprehensive design guide for building a **scalable, secure, and reproducible High-Performance Computing (HPC) environment** on Amazon Web Services (AWS), specifically tailored for **Genomic Nextflow workflows**.

The core of the infrastructure is built using **AWS ParallelCluster**, managed as Infrastructure as Code (IaC) via **Terraform**.

**Published documentation:** https://naratech-platforms.gitbook.io/genomics-nf-hpc-on-aws-parallelcluster/

- **Production-grade genomic variant discovery pipeline**
- **AWS ParallelCluster with SLURM scheduler**
- **CPU and GPU partitions for optimized workload execution**
- **Nextflow workflow orchestration**
- **Spack + Lmod for software management**
- **FSx for Lustre and EFS for high-performance storage**
- **Wazuh security monitoring on ECS Fargate**
- **Prometheus/Grafana observability stack**

## Architecture Diagram

```mermaid
flowchart TD
    subgraph "Public Internet"
        User((User))
    end

    subgraph "AWS Cloud - VPC"
        subgraph "Public Subnet"
            Bastion[Bastion Host <br/>or SSM/ VPN Gateway]
        end

        subgraph "Private Subnet"
            subgraph "Management Layer"
                HeadNode[Head Node<br/>SLURM Controller<br/>Nextflow Engine<br/>Spack/Lmod]
                WazuhMgr[Wazuh Manager<br/>Security Monitoring]
                PromGraf[Prometheus & Grafana<br/>Observability Stack]
            end

            subgraph "Compute Layer (Autoscaling)"
                subgraph "CPU Partition"
                    CPUNodes[c6a/c7i Instances<br/>Alignment, QC, Pre-processing]
                end
                subgraph "GPU Partition"
                    GPUNodes[g5/g6 Instances<br/>DeepVariant, GPU Acceleration]
                end
            end

            subgraph "Storage Layer"
                EFS[(Amazon EFS<br/>Home Dirs, Scripts)]
                FSx[(FSx for Lustre<br/>Scratch, High-perf I/O)]
                S3[(Amazon S3<br/>Raw Data, Long-term Storage)]
            end
        end
    end

    User -- r1 --> Bastion
    
    Bastion --> HeadNode
    HeadNode --> CPUNodes
    HeadNode --> GPUNodes
    CPUNodes --- FSx
    GPUNodes --- FSx
    HeadNode --- EFS
    CPUNodes --- EFS
    GPUNodes --- EFS
    FSx <--> S3
    
    WazuhMgr --> HeadNode
    WazuhMgr -.-> CPUNodes
    WazuhMgr -.-> GPUNodes
    PromGraf -.-> HeadNode
    PromGraf -.-> CPUNodes
    PromGraf -.-> GPUNodes

    linkStyle default stroke-width:2px,fill:none,stroke:#F472B6,stroke-dasharray: 5 5,animation:flow
    
    classDef default fill:#1F2937,stroke:#22D3EE,color:#E5E7EB,stroke-width:2px;
    classDef storage fill:#1F2937,stroke:#22D3EE,color:#E5E7EB,stroke-width:2px,stroke-dasharray: 5 5;
    class EFS,FSx,S3 storage;
```

## Quick Navigation

| Section | Description |
|---------|-------------|
| [Project Overview](docs/01-project-overview.md) | Objectives, design principles, and target audience |
| [System Architecture](docs/02-architecture.md) | Component breakdown and deployment model |
| [Technology Stack](docs/03-tech-stack.md) | Compute, storage, and software decisions |
| [Terraform Provisioning](docs/04-terraform.md) | Infrastructure as Code setup |
| [Workflow Design](docs/05-workflow.md) | Nextflow pipeline execution flow |
| [Security & Observability](docs/06-security-observability.md) | Wazuh, Prometheus, and Grafana integration |
| [Cost Optimization](docs/07-cost-optimization.md) | Strategies for minimizing TCO |
| [Conclusion](docs/08-conclusion.md) | Summary and future enhancements |
| [References](docs/09-references.md) | External resources and citations |
| [Developer Guidance](docs/10-developer-guidance.md) | SSM access, SLURM, and GPU management |
| [Troubleshooting](docs/11-troubleshooting-gpu.md) | GPU and CUDA issue resolution |
| [Validation Checklist](docs/12-validation-checklist.md) | Post-deployment verification steps |
| [Ansible Post-Provisioning](docs/13-ansible.md) | Post-provision configuration and Lmod setup |

## Getting Started

1. Review the [Project Overview](docs/01-project-overview.md) to understand the goals
2. Study the [System Architecture](docs/02-architecture.md) for component understanding
3. Follow the [Terraform Provisioning](docs/04-terraform.md) guide to deploy infrastructure
4. Use the [Validation Checklist](docs/12-validation-checklist.md) to verify deployment

## Repository Structure

```
hpc-genomics-nf/
├── README.md            # Project overview (this file)
├── docs/                # GitBook documentation
├── ansible/             # Post-provision configuration
├── terraform/           # Infrastructure as Code
├── nextflow/            # Pipeline definitions
└── modules/             # Reusable components
```

