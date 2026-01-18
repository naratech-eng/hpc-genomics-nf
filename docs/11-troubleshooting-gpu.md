# Troubleshooting Guide: GPU and CUDA Issues

This guide addresses common issues developers and server managers may encounter when working with GPU resources, SLURM scheduling, and CUDA-dependent software managed by Spack/Lmod.

---

## SLURM GPU Allocation Failures

The most frequent issues involve jobs failing to start or remaining in a `PENDING` state due to resource allocation errors.

### Issue Summary Table

| Symptom | Possible Cause | Server Manager Action | Developer Action |
| :--- | :--- | :--- | :--- |
| **Job PENDING: `Resources`** | Requested GPU resources not available | Check Grafana dashboard; verify autoscaling config | Verify correct partition; reduce GPU count if possible |
| **Job Fails: `Invalid generic resource (gres)`** | GRES syntax incorrect or not recognized | Confirm `GresTypes` in `/etc/slurm/slurm.conf` | Use exact syntax `--gres=gpu:N` |
| **Job PENDING: `ReqNodeNotAvail`** | Compute Node failed to launch | Check CloudWatch Logs for boot errors | Report to server manager with job ID |
| **Job Starts but Fails Immediately** | CUDA libraries not found | Verify GPU AMI has correct drivers | Load correct module before running |

---

### Detailed Troubleshooting

#### Job PENDING: Resources

**Symptom:** Job stays in `PENDING` state with reason `Resources`

**Diagnosis:**
```bash
# Check job details
squeue -j <job_id> --format="%i %j %t %r %S"

# Check partition availability
sinfo -p gpu

# Check resource requests
scontrol show job <job_id>
```

**Resolution:**
1. Check the **SLURM Workload Analysis** Grafana dashboard for queue length and node status
2. Verify ParallelCluster autoscaling is not hitting limits:
   ```yaml
   # Check max nodes in ParallelCluster config
   ComputeResources:
     - Name: gpu
       MaxCount: 4  # Increase if needed
   ```
3. Wait for resources to become available or reduce GPU count

#### Job Fails: Invalid generic resource (gres)

**Symptom:** Job rejected with error `Invalid generic resource (gres) specification`

**Diagnosis:**
```bash
# Check SLURM gres configuration
scontrol show config | grep GresTypes

# View node gres
scontrol show node <node_name> | grep Gres
```

**Resolution:**
- **Server Manager:** Confirm the SLURM configuration:
  ```bash
  # /etc/slurm/slurm.conf
  GresTypes=gpu
  
  # /etc/slurm/gres.conf
  NodeName=gpu-[1-4] Name=gpu File=/dev/nvidia[0-3]
  ```
- **Developer:** Use exact syntax `--gres=gpu:1` without model names unless specifically configured

#### Job PENDING: ReqNodeNotAvail

**Symptom:** Job pending with reason `ReqNodeNotAvail, Reserved for maintenance`

**Diagnosis:**
```bash
# Check CloudWatch Logs for ParallelCluster
aws logs tail /aws/parallelcluster/<cluster-name>/compute-fleet

# Check EC2 instance status
aws ec2 describe-instances --filters "Name=tag:Name,Values=*compute*"
```

**Resolution:**
1. Check CloudWatch Logs for node boot errors (NVIDIA driver, pcluster bootstrap)
2. Verify the GPU AMI is valid and accessible
3. Check AWS service quotas for the instance type

---

## CUDA and Lmod/Spack Module Issues

These issues typically occur after a job has successfully started on a GPU node, but the application fails to execute.

### Issue 1: libcuda.so Not Found

**Symptom:**
```
error while loading shared libraries: libcuda.so.1: cannot open shared object file: No such file or directory
```

**Root Cause:** The system's dynamic linker cannot find the NVIDIA driver libraries.

**Resolution:**

**Server Manager:**
```bash
# Verify driver installation
nvidia-smi

# Check library path
ls -la /usr/lib64/libcuda*

# If missing, ensure ldconfig includes driver path
cat /etc/ld.so.conf.d/nvidia.conf
# Should contain: /usr/lib64

# Regenerate library cache
sudo ldconfig
```

**Developer:**
```bash
# Ensure module is loaded
module load deepvariant/1.6-cuda-12.2

# Verify LD_LIBRARY_PATH
echo $LD_LIBRARY_PATH

# Manual workaround if needed
export LD_LIBRARY_PATH=/usr/lib64:$LD_LIBRARY_PATH
```

### Issue 2: CUDA Version Mismatch

**Symptom:**
```
CUDA driver version is insufficient for CUDA runtime version
```

**Root Cause:** Application compiled with newer CUDA than the installed driver supports.

**Diagnosis:**
```bash
# Check driver CUDA version
nvidia-smi
# Look for "CUDA Version: X.Y"

# Check application CUDA requirement
module show deepvariant/1.6-cuda-12.2
# Look for CUDA dependencies
```

**Resolution:**

**Server Manager:**
1. Check GPU Performance Monitor dashboard for driver version
2. Update GPU Compute Node AMI with newer driver if needed:
   ```bash
   # On the AMI build instance
   sudo yum install -y nvidia-driver-latest-dkms cuda-12-2
   ```

**Developer:**
```bash
# Install software for older CUDA version
spack install deepvariant ^cuda@11.8

# Load compatible module
module load deepvariant/1.6-cuda-11.8
```

### Issue 3: Spack Module Not Found

**Symptom:**
```
Lmod Error: Module 'deepvariant/1.6-cuda-12.2' not found.
```

**Root Cause:** Spack/Lmod integration not configured correctly.

**Diagnosis:**
```bash
# Check MODULEPATH
echo $MODULEPATH

# List available modules
module avail

# Check Spack module location
spack location -i deepvariant
ls $(spack location -i deepvariant)/../../../modules/
```

**Resolution:**

**Server Manager:**
```bash
# Verify MODULEPATH includes Spack modules
# Add to /etc/profile.d/spack.sh:
export MODULEPATH=$MODULEPATH:/opt/spack/share/spack/modules/linux-*

# Regenerate modules after Spack install
spack module lmod refresh --delete-tree
```

**Developer:**
```bash
# List all available modules
module avail 2>&1 | grep -i deepvariant

# Use exact module name
module spider deepvariant

# Load with full path if needed
module load /opt/spack/share/spack/modules/linux-amzn2-x86_64/deepvariant/1.6-cuda-12.2
```

---

## Diagnostic Commands Reference

### GPU Status

```bash
# NVIDIA driver and GPU status
nvidia-smi

# Detailed GPU info
nvidia-smi -q

# Watch GPU utilization
nvidia-smi dmon -s u

# Check CUDA version
nvcc --version
```

### SLURM Diagnostics

```bash
# Cluster overview
sinfo

# Detailed node info
scontrol show node <node_name>

# Job details
scontrol show job <job_id>

# Job accounting
sacct -j <job_id> --format=JobID,JobName,Partition,State,ExitCode,Elapsed

# GRES configuration
scontrol show config | grep -i gres
```

### Module System

```bash
# List loaded modules
module list

# Available modules
module avail

# Module details
module show <module_name>

# Search for module
module spider <keyword>
```

---

## Quick Reference: Common Fixes

| Problem | Quick Fix |
| :--- | :--- |
| GPU job stuck pending | Check `sinfo -p gpu` and autoscaling limits |
| GRES error | Use `--gres=gpu:1` exactly |
| libcuda.so missing | Run `sudo ldconfig` on compute node AMI |
| CUDA version mismatch | Match application CUDA to driver version |
| Module not found | Check `$MODULEPATH` includes Spack modules |
| Job fails silently | Check `sacct -j <job_id>` for exit code |

---

## Escalation Path

If issues persist after following this guide:

1. **Collect Diagnostics:**
   - Job ID and submission command
   - Output of `scontrol show job <job_id>`
   - Output of `nvidia-smi` (if accessible)
   - Relevant CloudWatch Logs

2. **Contact:**
   - Server Manager for infrastructure issues
   - Pipeline Developer for workflow issues
   - AWS Support for ParallelCluster/EC2 issues
