# PBS → Slurm Guide (English)

## What changed?

Our cluster switched from **PBS** (old) to **Slurm** (new).  
Everything works the same — just the command names changed.

Think of it like switching from one ride-sharing app to another. Same destination, different app.

---

## Cluster Overview

| Partition | Nodes | Purpose | Default |
|-----------|-------|---------|---------|
| `cpu` | cnode1, cnode2, cnode3 | CPU jobs | ⭐ YES (automatic if unspecified) |
| `gpu` | gnode1–gnode6 | GPU jobs | NO (need `-p gpu`) |
| `all` | all nodes | CPU on all nodes | NO |

Each gnode: 40 CPU, 4× V100 GPU (16GB), 187GB RAM  
Each cnode: 40 CPU, 188GB RAM

---

## Command Reference

| What you want to do | PBS (old) | Slurm (new) |
|---------------------|-----------|-------------|
| Submit a job | `qsub job.sh` | `sbatch job.sh` |
| Check job status | `qstat` | `squeue` |
| Cancel a job | `qdel 12345` | `scancel 12345` |
| Cancel all my jobs | `qdel -u $USER` | `scancel -u $USER` |
| Check cluster status | `pbsnodes` | `sinfo` |
| Check a specific node | `pbsnodes gnode1` | `scontrol show node gnode1` |
| Interactive session | `qsub -I` | `srun --pty bash` |
| Job details | `qstat -f 12345` | `scontrol show job 12345` |

---

## Submitting Jobs

### CPU job (default — goes to cnode automatically)

```bash
sbatch job.sh

# Specify CPU count
sbatch --cpus-per-task=20 job.sh

# Specify memory
sbatch --mem=50G job.sh
```

### GPU job (must specify `-p gpu` and `--gres`)

```bash
sbatch -p gpu --gres=gpu:2 job.sh

# 4 GPUs + 16 CPUs
sbatch -p gpu --gres=gpu:4 --cpus-per-task=16 job.sh
```

### Interactive GPU session

```bash
srun -p gpu --gres=gpu:2 --cpus-per-task=8 --pty bash

# Verify GPU inside the session:
nvidia-smi

# Always exit properly — closing terminal does NOT stop the job!
exit
```

---

## Updating Your Job Scripts

Replace `#PBS` headers with `#SBATCH`:

### CPU job

```bash
#!/bin/bash
#SBATCH --job-name=myjob
#SBATCH --output=%j.out        # %j = JOBID
#SBATCH --error=%j.err
#SBATCH --cpus-per-task=40
#SBATCH --mem=100G
#SBATCH --time=24:00:00        # format: HH:MM:SS

cd $SLURM_SUBMIT_DIR           # go to directory where sbatch was run
./myprogram
```

### GPU job

```bash
#!/bin/bash
#SBATCH --job-name=gpu_job
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p gpu
#SBATCH --gres=gpu:4
#SBATCH --cpus-per-task=16
#SBATCH --time=24:00:00

cd $SLURM_SUBMIT_DIR
python train.py
```

---

## Environment Variables

| Information | PBS (old) | Slurm (new) |
|-------------|-----------|-------------|
| Job ID | `$PBS_JOBID` | `$SLURM_JOB_ID` |
| Node list | `$PBS_NODEFILE` | `$SLURM_NODELIST` |
| CPU count | `$PBS_NCPUS` | `$SLURM_CPUS_PER_TASK` |
| Total processes | `$PBS_NP` | `$SLURM_NTASKS` |
| Current rank | — | `$SLURM_PROCID` |
| Submit directory | `$PBS_O_WORKDIR` | `$SLURM_SUBMIT_DIR` |
| GPU | `$CUDA_VISIBLE_DEVICES` | `$CUDA_VISIBLE_DEVICES` (same) |

---

## MPI Job Manual

### What is MPI?

MPI (Message Passing Interface) lets multiple CPUs across multiple computers work together on one calculation.  
Think of it like a factory where many workers collaborate — MPI handles how they divide work and communicate.

- **Rank**: each process has a number starting from 0
- **Node**: a computer participating in the job
- **Task**: Slurm's word for each MPI process

### srun vs mpirun

| | `srun` | `mpirun` |
|--|--------|----------|
| Recommended | ⭐ Yes | Works but legacy |
| Inter-node | Native support | Needs extra config |
| Resource tracking | Full Slurm tracking | May not track properly |
| GPU assignment | Automatic | Manual |

**Use `srun` instead of `mpirun` in Slurm environments.**

---

### Single-node MPI (multiple CPUs on one machine)

```bash
#!/bin/bash
#SBATCH --job-name=mpi_single
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -N 1                   # 1 node
#SBATCH --ntasks=40            # 40 MPI processes
#SBATCH --cpus-per-task=1      # 1 CPU per process

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### Multi-node MPI (multiple computers) — Most Important!

```bash
#!/bin/bash
#SBATCH --job-name=mpi_multi
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p cpu
#SBATCH -N 3                   # 3 nodes
#SBATCH --ntasks=12            # 12 total MPI processes
#SBATCH --ntasks-per-node=4    # 4 processes per node
#SBATCH --cpus-per-task=1

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### GPU + MPI Multi-node

```bash
#!/bin/bash
#SBATCH --job-name=gpu_mpi
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p gpu
#SBATCH -N 2                   # 2 gnodes
#SBATCH --ntasks=8             # 8 total MPI processes
#SBATCH --ntasks-per-node=4    # 4 per node (matches 4 GPUs)
#SBATCH --gres=gpu:4           # 4 GPUs per node
#SBATCH --cpus-per-task=4

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### Hybrid MPI + OpenMP

For programs that use both multi-node (MPI) and multi-thread (OpenMP):

```bash
#!/bin/bash
#SBATCH --job-name=hybrid
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p cpu
#SBATCH -N 3                   # 3 nodes
#SBATCH --ntasks=6             # 6 MPI processes (2 per node)
#SBATCH --ntasks-per-node=2
#SBATCH --cpus-per-task=20     # 20 OpenMP threads per MPI process

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### Verify MPI environment in your script

```bash
#!/bin/bash
#SBATCH -p cpu
#SBATCH -N 2
#SBATCH --ntasks=8

srun bash -c '
echo "Rank $SLURM_PROCID / Total $SLURM_NTASKS — running on $(hostname)"
'
```

Example output:
```
Rank 0 / Total 8 — running on cnode1
Rank 1 / Total 8 — running on cnode1
Rank 4 / Total 8 — running on cnode2
...
```

---

### MPI Environment Variables

| Variable | Description |
|----------|-------------|
| `$SLURM_NTASKS` | Total number of MPI processes |
| `$SLURM_PROCID` | Current process rank (starts at 0) |
| `$SLURM_NODELIST` | List of all nodes in the job |
| `$SLURM_NNODES` | Total number of nodes |
| `$SLURM_NTASKS_PER_NODE` | Processes per node |
| `$SLURM_CPUS_PER_TASK` | CPUs per process (for OpenMP) |

---

### MPI Troubleshooting

**Q: Inter-node job says "cannot find srun"**  
A: Fixed (2026-05-25). `/usr/bin/srun` is now installed on all compute nodes.

**Q: All MPI processes run on the same node**  
A: Make sure you specify both `-N <nodes>` and `--ntasks` larger than single-node capacity.

**Q: GPU MPI job can't see GPUs**  
A: Both `-p gpu` and `--gres=gpu:N` are required — don't skip either.

**Q: Must use mpirun instead of srun**  
```bash
# If you must use mpirun:
mpirun --mca btl_tcp_if_include eth0 -np 8 ./myprogram
# But srun is strongly recommended
```

---

## SSH Directly to Nodes

```bash
ssh gnode1   # GPU node (gnode1–gnode6)
ssh cnode1   # CPU node (cnode1–cnode3)
```

---

## Common Mistakes

### ❌ Forgot `-p gpu` for GPU jobs
```bash
# Wrong — job goes to CPU node, no GPU available
sbatch --gres=gpu:2 job.sh

# Correct
sbatch -p gpu --gres=gpu:2 job.sh
```

### ❌ Closing terminal to stop an interactive job
Closing the terminal does **NOT** stop `srun --pty bash` — it keeps running.  
Always use `exit` or `scancel <JOBID>`.

### ❌ Multi-node MPI without --ntasks
```bash
# Wrong — only 1 process per node
sbatch -N 3 job.sh

# Correct — explicitly set total process count
sbatch -N 3 --ntasks=12 job.sh
```

---

*Last updated: 2026-05-25*
