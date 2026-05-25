# PBS → Slurm 移行ガイド（日本語）

## なにが変わったの？

クラスターのジョブ管理システムが **PBS**（旧）から **Slurm**（新）に変わりました。  
やることは同じです。コマンドの名前が変わっただけです。

タクシーアプリが変わったようなものです。乗り場も目的地も同じ、アプリだけ違います。

---

## クラスター構成

| Partition | ノード | 用途 | デフォルト |
|-----------|--------|------|-----------|
| `cpu` | cnode1, cnode2, cnode3 | CPU ジョブ | ⭐ YES（指定なしで自動） |
| `gpu` | gnode1〜gnode7 | GPU ジョブ | NO（`-p gpu` が必要） |
| `all` | 全ノード | 全ノード CPU 利用 | NO |

各 gnode：CPU 40コア、V100 GPU × 4（16GB）、RAM 187GB  
各 cnode：CPU 40コア、RAM 188GB

---

## コマンド対照表

| やりたいこと | PBS（旧） | Slurm（新） |
|-------------|-----------|-------------|
| ジョブを投入する | `qsub job.sh` | `sbatch job.sh` |
| ジョブの状態を見る | `qstat` | `squeue` |
| ジョブをキャンセルする | `qdel 12345` | `scancel 12345` |
| 自分の全ジョブを削除 | `qdel -u $USER` | `scancel -u $USER` |
| ノードの状態を見る | `pbsnodes` | `sinfo` |
| 特定ノードの詳細 | `pbsnodes gnode1` | `scontrol show node gnode1` |
| 対話型セッション | `qsub -I` | `srun --pty bash` |
| ジョブの詳細 | `qstat -f 12345` | `scontrol show job 12345` |

---

## ジョブの投入

### CPU ジョブ（デフォルト → cnode に自動配置）

```bash
sbatch job.sh

# CPU 数を指定
sbatch --cpus-per-task=20 job.sh

# メモリを指定
sbatch --mem=50G job.sh
```

### GPU ジョブ（`-p gpu` と `--gres` が必須）

```bash
sbatch -p gpu --gres=gpu:2 job.sh

# GPU 4枚 + CPU 16コア
sbatch -p gpu --gres=gpu:4 --cpus-per-task=16 job.sh
```

---

## スクリプトの書き換え方

`#PBS` を `#SBATCH` に書き換えるだけです。

### CPU ジョブ

```bash
#!/bin/bash
#SBATCH --job-name=myjob
#SBATCH --output=%j.out        # %j は JOBID に自動で置き換わる
#SBATCH --error=%j.err
#SBATCH --cpus-per-task=40
#SBATCH --mem=100G
#SBATCH --time=24:00:00        # 形式：時:分:秒

cd $SLURM_SUBMIT_DIR           # sbatch を実行したディレクトリに移動
./myprogram
```

### GPU ジョブ

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

## 環境変数の対照表

| 知りたい情報 | PBS（旧） | Slurm（新） |
|-------------|-----------|-------------|
| ジョブ ID | `$PBS_JOBID` | `$SLURM_JOB_ID` |
| ノード一覧 | `$PBS_NODEFILE` | `$SLURM_NODELIST` |
| CPU 数 | `$PBS_NCPUS` | `$SLURM_CPUS_PER_TASK` |
| 総プロセス数 | `$PBS_NP` | `$SLURM_NTASKS` |
| 現在のランク | — | `$SLURM_PROCID` |
| 投入ディレクトリ | `$PBS_O_WORKDIR` | `$SLURM_SUBMIT_DIR` |
| GPU | `$CUDA_VISIBLE_DEVICES` | `$CUDA_VISIBLE_DEVICES`（同じ）|

---

## MPI ジョブ 完全マニュアル

### MPI とは？

MPI（Message Passing Interface）は、複数の CPU や複数のコンピュータが協力して計算する技術です。  
工場に多くの作業員がいて、MPI が仕事の割り振りと連絡を担当するようなイメージです。

- **Rank**：各プロセスに割り当てられた番号（0 始まり）
- **Node**：計算に参加するコンピュータ
- **Task**：Slurm での MPI プロセスの呼び方

### srun vs mpirun

| | `srun` | `mpirun` |
|--|--------|----------|
| 推奨度 | ⭐ 推奨 | 使えるが旧式 |
| ノード間通信 | ネイティブ対応 | 追加設定が必要 |
| リソース追跡 | Slurm が完全管理 | 一部追跡できない場合あり |
| GPU 割り当て | 自動で正確 | 手動設定が必要 |

**Slurm 環境では `mpirun` の代わりに `srun` を使ってください。**

---

### 単一ノード MPI（1台のコンピュータの複数 CPU）

```bash
#!/bin/bash
#SBATCH --job-name=mpi_single
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -N 1                   # ノード 1台
#SBATCH --ntasks=40            # MPI プロセス 40個
#SBATCH --cpus-per-task=1      # プロセスあたり CPU 1個

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### マルチノード MPI（複数コンピュータの協調計算） — 最重要！

```bash
#!/bin/bash
#SBATCH --job-name=mpi_multi
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p cpu
#SBATCH -N 3                   # 3ノード使用
#SBATCH --ntasks=12            # 合計 12 MPI プロセス
#SBATCH --ntasks-per-node=4    # 1ノードあたり 4プロセス
#SBATCH --cpus-per-task=1

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### GPU + MPI マルチノード

```bash
#!/bin/bash
#SBATCH --job-name=gpu_mpi
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p gpu
#SBATCH -N 2                   # gnode 2台
#SBATCH --ntasks=8             # 合計 8 MPI プロセス
#SBATCH --ntasks-per-node=4    # 1ノードあたり 4（GPU 4枚に対応）
#SBATCH --gres=gpu:4           # 1ノードあたり GPU 4枚
#SBATCH --cpus-per-task=4

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### MPI + OpenMP ハイブリッド並列

マルチノード（MPI）とマルチスレッド（OpenMP）を同時に使うプログラム向け：

```bash
#!/bin/bash
#SBATCH --job-name=hybrid
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p cpu
#SBATCH -N 3                   # 3ノード
#SBATCH --ntasks=6             # 合計 6 MPI プロセス（1ノード 2個）
#SBATCH --ntasks-per-node=2
#SBATCH --cpus-per-task=20     # 1 MPI プロセスあたり 20 OpenMP スレッド

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### スクリプト内で MPI 環境を確認する

```bash
#!/bin/bash
#SBATCH -p cpu
#SBATCH -N 2
#SBATCH --ntasks=8

srun bash -c '
echo "Rank $SLURM_PROCID / Total $SLURM_NTASKS — $(hostname) で実行中"
'
```

出力例：
```
Rank 0 / Total 8 — cnode1 で実行中
Rank 1 / Total 8 — cnode1 で実行中
Rank 4 / Total 8 — cnode2 で実行中
...
```

---

### MPI 関連の環境変数

| 変数 | 説明 |
|------|------|
| `$SLURM_NTASKS` | MPI プロセスの合計数 |
| `$SLURM_PROCID` | 現在のプロセスのランク（0 始まり）|
| `$SLURM_NODELIST` | ジョブに参加している全ノードのリスト |
| `$SLURM_NNODES` | ノード総数 |
| `$SLURM_NTASKS_PER_NODE` | 1ノードあたりのプロセス数 |
| `$SLURM_CPUS_PER_TASK` | 1プロセスあたりの CPU 数（OpenMP 用）|

---

### MPI トラブルシューティング

**Q: ノード間ジョブで「srun が見つからない」エラーが出る**  
A: 修正済み（2026-05-25）。全計算ノードに `/usr/bin/srun` をインストール済みです。

**Q: MPI プロセスが全部同じノードで動いてしまう**  
A: `-N ノード数` と `--ntasks` の両方を正しく指定してください。

**Q: GPU MPI ジョブで GPU が見えない**  
A: `-p gpu` と `--gres=gpu:N` の両方が必須です。片方だけでは動きません。

**Q: どうしても mpirun を使いたい場合**  
```bash
mpirun --mca btl_tcp_if_include eth0 -np 8 ./myprogram
# ただし srun を強く推奨します
```

---

## Conda 環境の使い方

> **注意**：sbatch スクリプト内で `conda activate` を直接使うことはできません。sbatch は非対話型シェルで動くため、conda init が読み込まれません。以下の方法を使ってください。

### 方法 1：source activate（推奨、sbatch スクリプト用）

```bash
#!/bin/bash
#SBATCH -p gpu
#SBATCH --gres=gpu:2
#SBATCH --output=%j.out

source /path/to/your/miniforge3/bin/activate 環境名
python train.py
```

### 方法 2：conda run（コマンドラインで手軽に）

```bash
srun -p gpu --gres=gpu:2 conda run -n 環境名 python train.py
```

### 方法 3：フルパスで直接指定（最も確実）

```bash
srun -p gpu --gres=gpu:2 /path/to/miniforge3/envs/環境名/bin/python train.py
```

### 自分の conda パスを調べる方法

```bash
which conda
# 例：/home/username/miniforge3/bin/conda
# activate パス：/home/username/miniforge3/bin/activate
```

---

## よくある間違い

### ❌ GPU ジョブで `-p gpu` を忘れる
```bash
# 間違い — CPU ノードに投入されて GPU が使えない
sbatch --gres=gpu:2 job.sh

# 正しい
sbatch -p gpu --gres=gpu:2 job.sh
```

### ❌ マルチノード MPI で --ntasks を忘れる
```bash
# 間違い — ノードごとに 1プロセスしか動かない
sbatch -N 3 job.sh

# 正しい — 総プロセス数を明示する
sbatch -N 3 --ntasks=12 job.sh
```

---

*最終更新：2026-05-25*
