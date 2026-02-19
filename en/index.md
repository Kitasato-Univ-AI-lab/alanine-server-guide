[日本語版](/alanine-server-guide/)

# alanine Server: SSH Access and SGE Job Submission Guide

## 1. SSH Access

### 1.1 Connection Targets

- Hostname: `alanine`
- Global IP: `202.251.201.250`
- Local IP: `192.168.0.6`
- SSH ports: both `22` and `7522`
- Authentication: public key only (`PasswordAuthentication no`)
- Root login: disabled (`PermitRootLogin no`)

### 1.2 Prerequisites (Client PC)

1. Create a key if you do not have one.

```bash
ssh-keygen -t ed25519 -C "your_name@your_pc"
```

2. Send your public key to the administrator and have it added to server-side `~/.ssh/authorized_keys`.

Check your public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

### 1.3 Minimal Connection Commands

```bash
ssh -p 22 <username>@202.251.201.250
```

Or:

```bash
ssh -p 7522 <username>@202.251.201.250
```

### 1.4 Recommended `~/.ssh/config`

```sshconfig
Host alanine
  HostName 202.251.201.250
  User <username>
  Port 22
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  ServerAliveInterval 30
  ServerAliveCountMax 3
```

Then connect with:

```bash
ssh alanine
```

### 1.5 First Login Checks

```bash
hostname
whoami
pwd
```

Expected:

- `hostname` is `alanine`
- `whoami` is your username

### 1.6 Troubleshooting SSH Access

1. Check detailed logs first.

```bash
ssh -vvv -p 22 <username>@202.251.201.250
```

2. If you get `Permission denied (publickey)`:

- Verify your key exists in `authorized_keys`
- Verify permissions: `~/.ssh` is `700`, `authorized_keys` is `600`
- Verify loaded keys with `ssh-add -l`

3. If it times out:

- Try port `7522`
- Check firewall/network policy

## 2. Job Scheduler

This environment uses **SGE (Sun Grid Engine 8.1.9)**.

Main commands:

- Submit: `qsub`
- Status: `qstat`
- Cancel: `qdel`
- Host info: `qhost`
- Config: `qconf`

## 3. Queue Layout (Observed)

Available queues:

- `main.q` (CPU jobs)
- `gpu.q` (GPU jobs)
- `login.q` (interactive)

Parallel environments (PE):

- `smp`
- `mpi`

Host examples:

- `main.q`: `hpc52 hpc53 hpc54 hpc55 hpc56 hpc57`
- `gpu.q`: `hpc50 hpc51 hpc58`

Check commands:

```bash
qconf -sql
qstat -f
qhost
```

## 4. GPU Server Specs (Observed)

### 4.1 `qsub` GPU Nodes

| Host | CPU threads (`nproc`) | Memory | GPU |
|---|---:|---:|---|
| `hpc51` | 40 | ~503 GiB | Tesla P100-SXM2-16GB x8 |
| `hpc58` | 96 | ~1.0 TiB | A100-PCIE-40GB x8 |
| `hpc50` | 40 | ~256 GiB (`qconf`) | `gpu=3` (`qconf`) |

Notes:

- `hpc50` was unreachable during one check, so `qconf` values are used.
- Current `gpu.q` members are `hpc50 hpc51 hpc58`.

### 4.2 `qsub`-external GPU servers

| Host | CPU threads (`nproc`) | Memory | GPU |
|---|---:|---:|---|
| `hpch1` | 32 | ~1.0 TiB | NVIDIA H100 NVL x4 |
| `hpca1` | 64 | ~125 GiB | NVIDIA A100 80GB PCIe x1 |

## 5. Minimal Batch Job (CPU)

### 5.1 Script Example

`job_cpu.sh`:

```bash
#!/bin/bash
#$ -S /bin/bash
#$ -cwd
#$ -N test_cpu
#$ -q main.q
#$ -pe smp 4
#$ -l h_vmem=4G
#$ -j y
#$ -o logs/$JOB_NAME.$JOB_ID.log

set -eu
mkdir -p logs

echo "HOST=$(hostname)"
echo "USER=$(whoami)"
echo "START=$(date)"

python3 -V

sleep 10
echo "END=$(date)"
```

### 5.2 Submit

```bash
qsub job_cpu.sh
```

### 5.3 Monitor

```bash
qstat -u $USER
qstat -j <JOB_ID>
```

## 6. GPU Job Submission

Use `-q gpu.q` and `-l gpu=<count>` for GPU jobs.
In practice, people also pin nodes with `-l h=<node>` and align `CUDA_VISIBLE_DEVICES`.

`job_gpu.sh`:

```bash
#!/bin/bash
#$ -S /bin/bash
#$ -cwd
#$ -N test_gpu
#$ -q gpu.q
#$ -pe smp 8
#$ -l h_vmem=8G,gpu=1
#$ -j y
#$ -o logs/$JOB_NAME.$JOB_ID.log

set -eu
mkdir -p logs

export CUDA_VISIBLE_DEVICES="0"
hostname
nvidia-smi
python3 train.py
```

Submit:

```bash
qsub job_gpu.sh
```

### 6.1 Node-Pinned GPU Job Example

```bash
#!/bin/bash
#$ -S /bin/bash
#$ -cwd
#$ -N train_gpu_fixed
#$ -q gpu.q
#$ -l h=hpc58
#$ -l h_vmem=200G,gpu=2
#$ -j y
#$ -o logs/$JOB_NAME.$JOB_ID.log

set -eu
mkdir -p logs
export CUDA_VISIBLE_DEVICES="0,1"

source $HOME/.pyenv/versions/anaconda3-2023.03/bin/activate
conda activate <env_name>
python Train.py
```

## 7. Interactive Execution

### 7.1 `qlogin`

```bash
qlogin -q login.q
```

### 7.2 `qrsh`

```bash
qrsh -q main.q hostname
```

## 8. Useful Operations

### 8.1 List your jobs

```bash
qstat -u $USER
```

### 8.2 Show queue state

```bash
qstat -f
```

### 8.3 Show job details

```bash
qstat -j <JOB_ID>
```

### 8.4 Delete a job

```bash
qdel <JOB_ID>
```

### 8.5 Pre-submit validation

```bash
qsub -w v -q main.q -pe smp 4 -l h_vmem=4G -b y /bin/true
qsub -w v -q gpu.q  -pe smp 8 -l h_vmem=8G,gpu=1 -b y /bin/true
```

If you see `verification: found suitable queue(s)`, the request is schedulable.

## 9. Array Jobs

`job_array.sh`:

```bash
#!/bin/bash
#$ -S /bin/bash
#$ -cwd
#$ -N array_test
#$ -q main.q
#$ -pe smp 1
#$ -l h_vmem=2G
#$ -t 1-100
#$ -j y
#$ -o logs/$JOB_NAME.$JOB_ID.$TASK_ID.log

set -eu
mkdir -p logs

echo "TASK_ID=$SGE_TASK_ID"
python3 run_task.py --index "$SGE_TASK_ID"
```

Submit:

```bash
qsub job_array.sh
```

## 10. Recommended Practices

- Always use `-cwd`.
- Always define logs with `-j y` and `-o`.
- Always define memory with `h_vmem`.
- For GPU jobs, always request `gpu` resources.
- Use `qsub` for long-running workloads.

## 11. Common Issues

### 11.1 Job stays in `qw`

Check in this order:

1. `qstat -j <JOB_ID>` scheduling info
2. `qstat -f` queue state
3. Resource requests (`h_vmem`, `gpu`, `-pe smp`)
4. `qsub -w v ...` validation

### 11.2 GPU behavior is not as expected

Check in this order:

1. `qsub` flags (`-q gpu.q`, `-l gpu=<n>`, `-l h_vmem=...`)
2. Add `-l h=<node>` if node pinning is required
3. Print `nvidia-smi` and `torch.cuda.device_count()` in the job
4. Match `CUDA_VISIBLE_DEVICES` count with `gpu=<n>`

## 12. Quick Templates

### 12.1 Quick CPU

```bash
qsub -cwd -N quick_cpu -q main.q -pe smp 1 -l h_vmem=2G -b y '/bin/bash -lc "hostname; date"'
```

### 12.2 Quick GPU

```bash
qsub -cwd -N quick_gpu -q gpu.q -pe smp 1 -l h_vmem=4G,gpu=1 -b y '/bin/bash -lc "hostname; nvidia-smi -L"'
```

### 12.3 Practical GPU Training Template

```bash
#$ -S /bin/bash
#$ -cwd
#$ -j y
#$ -l h=hpc58
#$ -l h_vmem=200g
#$ -l gpu=2
export CUDA_VISIBLE_DEVICES="4,5"

source $HOME/.pyenv/versions/anaconda3-2023.03/bin/activate
conda activate <env_name>
python Train.py
```

## 13. CUDA Operations

### 13.1 Check CUDA in job context

```bash
which nvcc
nvcc --version
nvidia-smi
```

Notes:

- `nvcc --version`: `Cuda compilation tools, release 10.1, V10.1.243`
- `nvidia-smi`: Driver `460.73.01`, CUDA Version `11.2`
- Confirm framework/CUDA compatibility for your environment.

### 13.2 Recommended job header

```bash
#!/bin/bash
#$ -S /bin/bash
#$ -cwd
#$ -q gpu.q
#$ -pe smp 4
#$ -l h_vmem=8G,gpu=1

set -eu

echo "HOST=$(hostname)"
which nvcc
nvcc --version
nvidia-smi -L
```

### 13.3 Python (PyTorch) checks

- Run GPU checks inside `qsub` jobs.
- Isolate dependencies via `venv` or `conda`.

```bash
python - <<'PY'
import torch
print("torch:", torch.__version__)
print("cuda available:", torch.cuda.is_available())
print("device count:", torch.cuda.device_count())
PY
```

## 14. How to Use `hpch1` / `hpca1` (Direct Execution)

`hpch1` and `hpca1` are used through direct SSH execution.

If you are already logged in to `alanine`, you can directly enter `hpca1` with:

```bash
ssh hpca1
```

### 14.1 Connect and verify

```bash
ssh <username>@hpch1
hostname
nvidia-smi -L
```

For `hpca1`:

```bash
ssh <username>@hpca1
hostname
nvidia-smi -L
```

### 14.2 Run with `tmux`

```bash
tmux new -s train
source $HOME/.pyenv/versions/anaconda3-2023.03/bin/activate
conda activate <env_name>
export CUDA_VISIBLE_DEVICES="0,1,2,3"
python Train.py
```

On `hpca1` (single GPU), use for example:

```bash
tmux new -s train
source $HOME/.pyenv/versions/anaconda3-2023.03/bin/activate
conda activate <env_name>
export CUDA_VISIBLE_DEVICES="0"
python Train.py
```

Detach:

```bash
Ctrl-b d
```

Reattach:

```bash
tmux attach -t train
```
