# Terraform-Based Cluster Provisioning

This project adheres to the principle of **Infrastructure as Code (IaC)** by using **Terraform** to provision and manage the entire AWS environment, including the HPC cluster and the decoupled management plane.

---

## Purpose and Scope

Terraform is used to define the desired state of the infrastructure in declarative configuration files. This ensures that the environment is reproducible, version-controlled, and can be deployed consistently across different AWS regions or accounts.

### Terraform Provisions

- **Networking:** VPC, subnets (public and private), route tables, and security groups for the HPC cluster and the ECS Fargate services.
- **HPC Cluster:** The AWS ParallelCluster configuration itself, including instance types, scaling limits, and custom bootstrap actions.
- **Storage:** Amazon EFS and FSx for Lustre file systems, including their mounting and data-repository-association to S3.
- **Management Plane:** ECS Cluster, Fargate Task Definitions, Services for Wazuh SIEM Manager, Prometheus, and Grafana, along with the Application Load Balancer (ALB) and DNS records (Route53/ACM).
- **Security:** IAM roles and policies for all EC2 instances and ECS tasks, ensuring least-privilege access.

---

## Terraform Workflow

The standard Terraform workflow is used for managing the infrastructure lifecycle:

| Step | Command | Description |
| :--- | :--- | :--- |
| **Initialize** | `terraform init` | Prepares the working directory, downloads necessary providers, and initializes the backend for state storage. |
| **Plan** | `terraform plan` | Generates an execution plan, showing exactly what actions (create, update, destroy) Terraform will take to reach the desired state. **Crucial for review before applying.** |
| **Apply** | `terraform apply` | Executes the actions proposed in the plan, provisioning or updating the infrastructure resources. |
| **Destroy** | `terraform destroy` | Tears down all resources managed by the configuration. **Used to control costs when the cluster is not in use.** |

### Workflow Diagram

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  terraform init │ ──► │  terraform plan │ ──► │ terraform apply │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
                                                        ▼
                                               ┌─────────────────┐
                                               │   AWS Resources │
                                               │   Provisioned   │
                                               └─────────────────┘
                                                        │
                                                        ▼
                                               ┌─────────────────┐
                                               │terraform destroy│
                                               │ (Cost Control)  │
                                               └─────────────────┘
```

---

## Terraform Repository Structure

The project's repository structure is designed for modularity and clarity, separating the core infrastructure components. This structure uses Terraform modules to encapsulate and reuse configuration blocks.

```
terraform/
├── main.tf             # Root configuration: Defines providers, backend, and calls modules
├── variables.tf        # Input variables (e.g., region, instance types, VPC CIDR)
├── outputs.tf          # Output values (e.g., Grafana URL, Head Node IP, Cluster ARN)
├── modules/
│   ├── vpc/            # Module for creating the base VPC, subnets, and security groups
│   ├── parallelcluster/ # Module for deploying the AWS ParallelCluster resource
│   ├── storage/        # Module for creating EFS and FSx for Lustre
│   └── management/     # Module for deploying ECS Cluster, Fargate services
└── environments/       # Environment-specific variable files
    ├── dev.tfvars      # Variables for development environment
    └── prod.tfvars     # Variables for production environment
```

---

## Module Interaction

The `parallelcluster` module is the core of the HPC deployment. It relies on outputs from other modules to configure the cluster correctly.

| Input to `parallelcluster` Module | Source Module | Purpose |
| :--- | :--- | :--- |
| `vpc_id`, `subnet_id` | `vpc` | Specifies where the Head Node and Compute Nodes will be deployed. |
| `efs_id`, `fsx_id` | `storage` | Provides the File System IDs for automatic mounting on all cluster nodes. |
| `iam_role_arn` | `iam` (implicit) | Specifies the IAM role for the Head Node and Compute Nodes, which includes permissions for SSM, S3, and ParallelCluster operations. |
| `custom_ami_id` | `ami` (implicit) | Specifies the custom AMI (e.g., with pre-installed NVIDIA drivers) for the GPU partition. |

### Module Dependency Flow

```
┌─────────────┐
│  vpc module │
└──────┬──────┘
       │ vpc_id, subnet_id
       ▼
┌─────────────────┐     ┌────────────────┐
│ storage module  │     │  iam module    │
└───────┬─────────┘     └───────┬────────┘
        │ efs_id, fsx_id        │ iam_role_arn
        └───────────┬───────────┘
                    ▼
         ┌─────────────────────┐
         │ parallelcluster     │
         │ module              │
         └─────────────────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │ management module   │
         │ (ECS Fargate)       │
         └─────────────────────┘
```

This modular design ensures that changes to the network (e.g., changing the VPC CIDR) or the storage (e.g., changing FSx throughput) can be managed independently without modifying the core ParallelCluster configuration logic.

---

## Key Configuration Examples

### VPC Module Variables

```hcl
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "List of availability zones"
  default     = ["us-east-1a", "us-east-1b"]
}
```

### ParallelCluster Module Variables

```hcl
variable "cpu_instance_type" {
  description = "Instance type for CPU partition"
  default     = "c6a.4xlarge"
}

variable "gpu_instance_type" {
  description = "Instance type for GPU partition"
  default     = "g5.2xlarge"
}

variable "max_cpu_nodes" {
  description = "Maximum number of CPU compute nodes"
  default     = 10
}

variable "max_gpu_nodes" {
  description = "Maximum number of GPU compute nodes"
  default     = 4
}
```

---

## Best Practices

1. **State Management:** Use remote state storage (S3 + DynamoDB) for team collaboration
2. **Variable Files:** Use `.tfvars` files for environment-specific configurations
3. **Outputs:** Export important values (URLs, IPs, ARNs) for downstream use
4. **Tagging:** Apply consistent tags for cost tracking and resource management
5. **Documentation:** Comment complex configurations for maintainability
