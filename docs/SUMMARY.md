# Summary

* [Introduction](../README.md)

## Getting Started

* [Project Overview](01-project-overview.md)
* [System Architecture](02-architecture.md)
  * [Architecture Diagram](02-architecture.md#high-level-architecture-diagram)
  * [Component Breakdown](02-architecture.md#component-breakdown)
  * [Management Plane Deployment](02-architecture.md#management-plane-deployment-ecs-fargate)

## Design & Implementation

* [Technology Stack & Design Decisions](03-tech-stack.md)
  * [Compute and Scheduling](03-tech-stack.md#compute-and-scheduling)
  * [Storage Architecture](03-tech-stack.md#storage-architecture)
  * [Workflow Orchestration](03-tech-stack.md#workflow-orchestration)
  * [Software Management](03-tech-stack.md#software-management)
* [Terraform-Based Cluster Provisioning](04-terraform.md)
  * [Terraform Workflow](04-terraform.md#terraform-workflow)
  * [Repository Structure](04-terraform.md#terraform-repository-structure)
* [System Design Flow](05-workflow.md)

## Operations

* [Security and Observability](06-security-observability.md)
  * [Security Architecture](06-security-observability.md#security-architecture)
  * [Monitoring and Observability](06-security-observability.md#monitoring-and-observability)
* [Cost Optimization Strategies](07-cost-optimization.md)

## Guides

* [Developer and Server Manager Guidance](10-developer-guidance.md)
  * [Secure Cluster Access via SSM](10-developer-guidance.md#secure-cluster-access-via-aws-systems-manager-ssm)
  * [Node Roles and Responsibilities](10-developer-guidance.md#node-roles-and-responsibilities)
  * [Controlling CPU and GPU Resources](10-developer-guidance.md#controlling-cpu-and-gpu-resources-via-slurm)
* [Troubleshooting Guide: GPU and CUDA Issues](11-troubleshooting-gpu.md)
  * [SLURM GPU Allocation Failures](11-troubleshooting-gpu.md#slurm-gpu-allocation-failures)
  * [CUDA and Lmod/Spack Module Issues](11-troubleshooting-gpu.md#cuda-and-lmodspack-module-issues)
* [Post-Deployment Validation Checklist](12-validation-checklist.md)
* [Ansible Post-Provisioning](13-ansible.md)
* [CI/CD & Automation](14-ci-cd.md)

## Appendix

* [Conclusion and Future Work](08-conclusion.md)
* [References](09-references.md)
