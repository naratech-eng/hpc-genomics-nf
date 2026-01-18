# Summary

* [Introduction](README.md)

## Getting Started

* [Project Overview](docs/01-project-overview.md)
* [System Architecture](docs/02-architecture.md)
  * [Architecture Diagram](docs/02-architecture.md#high-level-architecture-diagram)
  * [Component Breakdown](docs/02-architecture.md#component-breakdown)
  * [Management Plane Deployment](docs/02-architecture.md#management-plane-deployment-ecs-fargate)

## Design & Implementation

* [Technology Stack & Design Decisions](docs/03-tech-stack.md)
  * [Compute and Scheduling](docs/03-tech-stack.md#compute-and-scheduling)
  * [Storage Architecture](docs/03-tech-stack.md#storage-architecture)
  * [Workflow Orchestration](docs/03-tech-stack.md#workflow-orchestration)
  * [Software Management](docs/03-tech-stack.md#software-management)
* [Terraform-Based Cluster Provisioning](docs/04-terraform.md)
  * [Terraform Workflow](docs/04-terraform.md#terraform-workflow)
  * [Repository Structure](docs/04-terraform.md#terraform-repository-structure)
* [System Design Flow](docs/05-workflow.md)

## Operations

* [Security and Observability](docs/06-security-observability.md)
  * [Security Architecture](docs/06-security-observability.md#security-architecture)
  * [Monitoring and Observability](docs/06-security-observability.md#monitoring-and-observability)
* [Cost Optimization Strategies](docs/07-cost-optimization.md)

## Guides

* [Developer and Server Manager Guidance](docs/10-developer-guidance.md)
  * [Secure Cluster Access via SSM](docs/10-developer-guidance.md#secure-cluster-access-via-aws-systems-manager-ssm)
  * [Node Roles and Responsibilities](docs/10-developer-guidance.md#node-roles-and-responsibilities)
  * [Controlling CPU and GPU Resources](docs/10-developer-guidance.md#controlling-cpu-and-gpu-resources-via-slurm)
* [Troubleshooting Guide: GPU and CUDA Issues](docs/11-troubleshooting-gpu.md)
  * [SLURM GPU Allocation Failures](docs/11-troubleshooting-gpu.md#slurm-gpu-allocation-failures)
  * [CUDA and Lmod/Spack Module Issues](docs/11-troubleshooting-gpu.md#cuda-and-lmodspack-module-issues)
* [Post-Deployment Validation Checklist](docs/12-validation-checklist.md)

## Appendix

* [Conclusion and Future Work](docs/08-conclusion.md)
* [References](docs/09-references.md)
