# Project Overview

**Date:** January 18, 2026  
**Target Audience:** HPC Learners, Bioinformatics Engineers, Cloud/

---
HPC Architects

This document outlines the design for a **scalable, secure, and reproducible High-Performance Computing (HPC) environment** on Amazon Web Services (AWS), specifically tailored for **Genomic Nextflow workflows**. The core of the infrastructure is built using **AWS ParallelCluster** [1], managed as Infrastructure as Code (IaC) via **Terraform**.

---

## Objective

The primary objective is to establish a production-grade genomic variant discovery pipeline capable of processing large-scale public sequencing data. The design prioritizes:

- **Scalability** through cloud-native autoscaling
- **Reproducibility** through containerization and versioned software management
- **Security** through a decoupled, containerized host-based intrusion detection and centralized auditing system

---

## Key Design Principles

| Principle | Description | Key Technology |
| :--- | :--- | :--- |
| **Infrastructure as Code (IaC)** | All infrastructure components are defined, provisioned, and managed through code for repeatability and version control. | Terraform, AWS ParallelCluster |
| **Scalability & Elasticity** | Compute resources automatically scale up and down based on workload demand, minimizing idle time and cost. | AWS ParallelCluster Autoscaling, SLURM |
| **Performance** | Utilize high-throughput storage and specialized compute instances (CPU/GPU) to accelerate I/O-intensive and computationally demanding stages. | FSx for Lustre, EC2 c6a/c7i, g5/g6 |
| **Reproducibility** | Software environments are isolated and managed to ensure consistent results across different runs and users. | Nextflow, Spack, Lmod |
| **Security & Compliance** | Implement host-based security monitoring and vulnerability management using a decoupled, containerized management plane. | Wazuh (ECS Fargate), Trivy, AWS IAM |

---

## Target Users

This documentation is designed for:

- **HPC Learners** - Those looking to understand cloud-based HPC concepts
- **Bioinformatics Engineers** - Practitioners building genomic pipelines
- **Cloud/HPC Architects** - Professionals designing scalable infrastructure

---

## Pipeline Capabilities

The genomic variant discovery pipeline performs:

1. **Quality Control** - FastQC analysis of raw sequencing data
2. **Alignment** - BWA-MEM alignment to reference genome
3. **Variant Calling** - GPU-accelerated DeepVariant
4. **Post-processing** - VCF filtering and annotation

---

## Data Sources

| Data Type | Source | Purpose |
| :--- | :--- | :--- |
| **Primary Data** | NCBI Sequence Read Archive (SRA), 1000 Genomes Project | Raw sequencing reads |
| **Reference Data** | GRCh38 reference genome (Ensembl/UCSC) | Alignment reference |
| **Benchmarking Data** | Genome in a Bottle (GIAB) | Validation and accuracy assessment |

---

## This covers:

- SLURM scheduling for CPU and GPU workloads
- Nextflow for workflow orchestration
- Software management using Spack and environment modules
- Infrastructure provisioning with Terraform
- Performance and cost optimization strategies
- Security and observability best practices

