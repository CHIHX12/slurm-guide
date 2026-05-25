# PBS → Slurm 使用指南（繁體中文）

## 這是什麼？

我們的電腦叢集從 **PBS**（舊系統）換成了 **Slurm**（新系統）。  
工作原理一樣，只是指令名稱不同。

把它想成：以前叫計程車用 A app，現在改用 B app。目的地一樣，只是叫車方式不同。

---

## 叢集架構

| Partition | 節點 | 用途 | 預設 |
|-----------|------|------|------|
| `cpu` | cnode1, cnode2, cnode3 | CPU 工作 | ⭐ YES（不指定自動來這） |
| `gpu` | gnode1〜gnode6 | GPU 工作 | NO（要加 `-p gpu`） |
| `all` | 全部節點 | 所有節點 CPU | NO |

每個 gnode：40 CPU、4× V100 GPU（16GB）、187GB RAM  
每個 cnode：40 CPU、188GB RAM

---

## 最常用指令對照

| 我想做的事 | PBS（舊） | Slurm（新） |
|-----------|-----------|-------------|
| 送出工作 | `qsub job.sh` | `sbatch job.sh` |
| 查看工作狀態 | `qstat` | `squeue` |
| 取消工作 | `qdel 12345` | `scancel 12345` |
| 取消自己所有工作 | `qdel -u $USER` | `scancel -u $USER` |
| 查看叢集狀態 | `pbsnodes` | `sinfo` |
| 查看特定節點 | `pbsnodes gnode1` | `scontrol show node gnode1` |
| 開互動式視窗 | `qsub -I` | `srun --pty bash` |
| 查看工作詳細 | `qstat -f 12345` | `scontrol show job 12345` |

---

## 送出工作

### CPU 工作（一般計算）

```bash
# 直接送出，系統自動分配到 cnode（CPU 節點）
sbatch job.sh

# 指定 CPU 數量
sbatch --cpus-per-task=20 job.sh

# 指定記憶體
sbatch --mem=50G job.sh
```

### GPU 工作（深度學習、CUDA）

```bash
# 一定要加 -p gpu 和 --gres=gpu:數量
sbatch -p gpu --gres=gpu:2 job.sh

# 使用 4 個 GPU + 16 個 CPU
sbatch -p gpu --gres=gpu:4 --cpus-per-task=16 job.sh
```

---

## 工作腳本怎麼改

把腳本開頭的 `#PBS` 換成 `#SBATCH`：

### CPU 工作

```bash
#!/bin/bash
#SBATCH --job-name=我的工作
#SBATCH --output=%j.out        # %j 會自動換成 JOBID
#SBATCH --error=%j.err
#SBATCH --cpus-per-task=40
#SBATCH --mem=100G
#SBATCH --time=24:00:00        # 格式：時:分:秒

cd $SLURM_SUBMIT_DIR           # 切換到投稿時所在的目錄
./我的程式
```

### GPU 工作

```bash
#!/bin/bash
#SBATCH --job-name=GPU工作
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

## 環境變數對照

| 想知道什麼 | PBS（舊） | Slurm（新） |
|-----------|-----------|-------------|
| 工作 ID | `$PBS_JOBID` | `$SLURM_JOB_ID` |
| 節點清單 | `$PBS_NODEFILE` | `$SLURM_NODELIST` |
| CPU 數量 | `$PBS_NCPUS` | `$SLURM_CPUS_PER_TASK` |
| 總 process 數 | `$PBS_NP` | `$SLURM_NTASKS` |
| 目前 process 編號 | — | `$SLURM_PROCID` |
| 投稿目錄 | `$PBS_O_WORKDIR` | `$SLURM_SUBMIT_DIR` |
| GPU | `$CUDA_VISIBLE_DEVICES` | `$CUDA_VISIBLE_DEVICES`（一樣）|

---

## MPI 工作完整手冊

### MPI 是什麼？

MPI（Message Passing Interface）是讓多個 CPU 或多台電腦同時合作計算的技術。  
就像一個工廠有很多工人，MPI 負責分配工作並讓大家互相溝通。

- **Rank**：每個 process 的編號，從 0 開始
- **Node**：參與計算的電腦節點
- **Task**：Slurm 中每個 MPI process 叫做一個 task

### srun vs mpirun

| | `srun` | `mpirun` |
|--|--------|----------|
| 推薦度 | ⭐ 推薦 | 可用但較舊 |
| 跨節點 | 原生支援 | 需額外設定 |
| 資源追蹤 | Slurm 完整追蹤 | 部分情況下無法追蹤 |
| GPU 分配 | 自動正確 | 需手動設定 |

**結論：Slurm 環境下請用 `srun` 取代 `mpirun`。**

---

### 單節點 MPI（同一台電腦的多個 CPU）

```bash
#!/bin/bash
#SBATCH --job-name=mpi_single
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -N 1                   # 使用 1 個節點
#SBATCH --ntasks=40            # 40 個 MPI process
#SBATCH --cpus-per-task=1      # 每個 process 用 1 個 CPU

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### 跨節點 MPI（多台電腦合作） — 最重要！

```bash
#!/bin/bash
#SBATCH --job-name=mpi_multi
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p cpu
#SBATCH -N 3                   # 使用 3 個節點
#SBATCH --ntasks=12            # 共 12 個 MPI process（每節點 4 個）
#SBATCH --ntasks-per-node=4    # 每個節點跑 4 個 process
#SBATCH --cpus-per-task=1

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### GPU + MPI 跨節點

```bash
#!/bin/bash
#SBATCH --job-name=gpu_mpi
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p gpu
#SBATCH -N 2                   # 使用 2 個 gnode
#SBATCH --ntasks=8             # 共 8 個 MPI process
#SBATCH --ntasks-per-node=4    # 每個節點 4 個 process（對應 4 個 GPU）
#SBATCH --gres=gpu:4           # 每個節點 4 個 GPU
#SBATCH --cpus-per-task=4

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### MPI + OpenMP 混合並行（Hybrid）

適合同時使用多節點（MPI）和多 thread（OpenMP）的程式：

```bash
#!/bin/bash
#SBATCH --job-name=hybrid
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p cpu
#SBATCH -N 3                   # 3 個節點
#SBATCH --ntasks=6             # 共 6 個 MPI process（每節點 2 個）
#SBATCH --ntasks-per-node=2
#SBATCH --cpus-per-task=20     # 每個 MPI process 用 20 個 thread（OpenMP）

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### 在腳本內確認 MPI 環境

```bash
#!/bin/bash
#SBATCH -p cpu
#SBATCH -N 2
#SBATCH --ntasks=8

srun bash -c '
echo "Rank $SLURM_PROCID / Total $SLURM_NTASKS — running on $(hostname)"
'
```

輸出範例：
```
Rank 0 / Total 8 — running on cnode1
Rank 1 / Total 8 — running on cnode1
Rank 4 / Total 8 — running on cnode2
...
```

---

### MPI 相關環境變數

| 變數 | 說明 |
|------|------|
| `$SLURM_NTASKS` | 總 MPI process 數 |
| `$SLURM_PROCID` | 目前 process 的 Rank（0 起算） |
| `$SLURM_NODELIST` | 所有參與節點清單 |
| `$SLURM_NNODES` | 節點總數 |
| `$SLURM_NTASKS_PER_NODE` | 每節點的 process 數 |
| `$SLURM_CPUS_PER_TASK` | 每個 process 的 CPU 數（OpenMP 用）|

---

### MPI 常見問題排查

**Q: 跨節點 job 出現「cannot find srun」**  
A: 已修復（2026-05-25）。各計算節點都已安裝 `/usr/bin/srun`。

**Q: MPI process 都跑在同一個節點**  
A: 確認有加 `-N 數量` 且 `--ntasks` 大於單節點 task 數。

**Q: GPU MPI job 看不到 GPU**  
A: 確認加了 `-p gpu` 和 `--gres=gpu:N`，缺一不可。

**Q: 想用 mpirun 而不是 srun**  
```bash
# 如果一定要用 mpirun，需要加這個參數：
mpirun --mca btl_tcp_if_include eth0 -np 8 ./myprogram
# 但強烈建議改用 srun
```

---

## 常見錯誤

### ❌ GPU 工作忘記加 `-p gpu`
```bash
# 錯：工作會跑到 CPU 節點，拿不到 GPU
sbatch --gres=gpu:2 job.sh

# 對：
sbatch -p gpu --gres=gpu:2 job.sh
```

### ❌ MPI 跨節點用 -N 但忘記設 --ntasks
```bash
# 這樣只會在每個節點跑 1 個 process
sbatch -N 3 job.sh

# 正確：明確指定 process 數
sbatch -N 3 --ntasks=12 job.sh
```

---

*更新日期：2026-05-25*
