# PBS → Slurm 마이그레이션 가이드 (한국어)

## 무엇이 바뀌었나요?

클러스터의 작업 관리 시스템이 **PBS**（구）에서 **Slurm**（신）으로 변경되었습니다.  
하는 일은 똑같습니다. 명령어 이름만 바뀌었습니다.

택시 앱이 바뀐 것과 같습니다. 타는 곳도 목적지도 같고, 앱만 달라진 것입니다.

---

## 클러스터 구성

| Partition | 노드 | 용도 | 기본값 |
|-----------|------|------|--------|
| `cpu` | cnode1, cnode2, cnode3 | CPU 작업 | ⭐ YES（지정 없으면 자동） |
| `gpu` | gnode1〜gnode7 | GPU 작업 | NO（`-p gpu` 필요） |
| `all` | 전체 노드 | 모든 노드 CPU 사용 | NO |

각 gnode：CPU 40코어, V100 GPU × 4（16GB）, RAM 187GB  
각 cnode：CPU 40코어, RAM 188GB

---

## 명령어 대조표

| 하고 싶은 일 | PBS (구) | Slurm (신) |
|-------------|----------|------------|
| 작업 제출 | `qsub job.sh` | `sbatch job.sh` |
| 작업 상태 확인 | `qstat` | `squeue` |
| 작업 취소 | `qdel 12345` | `scancel 12345` |
| 내 모든 작업 취소 | `qdel -u $USER` | `scancel -u $USER` |
| 클러스터 상태 확인 | `pbsnodes` | `sinfo` |
| 특정 노드 상세 확인 | `pbsnodes gnode1` | `scontrol show node gnode1` |
| 대화형 세션 | `qsub -I` | `srun --pty bash` |
| 작업 상세 정보 | `qstat -f 12345` | `scontrol show job 12345` |

---

## 작업 제출

### CPU 작업（기본값 → cnode 에 자동 배치）

```bash
sbatch job.sh

# CPU 수 지정
sbatch --cpus-per-task=20 job.sh

# 메모리 지정
sbatch --mem=50G job.sh
```

### GPU 작업（`-p gpu` 와 `--gres` 필수）

```bash
sbatch -p gpu --gres=gpu:2 job.sh

# GPU 4개 + CPU 16코어
sbatch -p gpu --gres=gpu:4 --cpus-per-task=16 job.sh
```

---

## 스크립트 수정 방법

`#PBS` 를 `#SBATCH` 로 바꾸면 됩니다.

### CPU 작업

```bash
#!/bin/bash
#SBATCH --job-name=myjob
#SBATCH --output=%j.out        # %j 는 JOBID 로 자동 치환
#SBATCH --error=%j.err
#SBATCH --cpus-per-task=40
#SBATCH --mem=100G
#SBATCH --time=24:00:00        # 형식：시:분:초

cd $SLURM_SUBMIT_DIR           # sbatch 를 실행한 디렉토리로 이동
./myprogram
```

### GPU 작업

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

## 환경 변수 대조표

| 알고 싶은 정보 | PBS (구) | Slurm (신) |
|--------------|----------|------------|
| 작업 ID | `$PBS_JOBID` | `$SLURM_JOB_ID` |
| 노드 목록 | `$PBS_NODEFILE` | `$SLURM_NODELIST` |
| CPU 수 | `$PBS_NCPUS` | `$SLURM_CPUS_PER_TASK` |
| 총 프로세스 수 | `$PBS_NP` | `$SLURM_NTASKS` |
| 현재 프로세스 번호 | — | `$SLURM_PROCID` |
| 제출 디렉토리 | `$PBS_O_WORKDIR` | `$SLURM_SUBMIT_DIR` |
| GPU | `$CUDA_VISIBLE_DEVICES` | `$CUDA_VISIBLE_DEVICES` (동일) |

---

## MPI 작업 완전 매뉴얼

### MPI 란 무엇인가요?

MPI（Message Passing Interface）는 여러 CPU 또는 여러 컴퓨터가 협력하여 계산하는 기술입니다.  
공장에 많은 작업자가 있고, MPI 가 작업 배분과 소통을 담당하는 것과 같습니다.

- **Rank**：각 프로세스에 부여된 번호（0 부터 시작）
- **Node**：계산에 참여하는 컴퓨터
- **Task**：Slurm 에서 MPI 프로세스를 부르는 말

### srun vs mpirun

| | `srun` | `mpirun` |
|--|--------|----------|
| 추천도 | ⭐ 추천 | 사용 가능하지만 구식 |
| 노드 간 통신 | 기본 지원 | 추가 설정 필요 |
| 리소스 추적 | Slurm 이 완전 관리 | 일부 추적 불가 |
| GPU 할당 | 자동으로 정확 | 수동 설정 필요 |

**Slurm 환경에서는 `mpirun` 대신 `srun` 을 사용하세요.**

---

### 단일 노드 MPI（한 컴퓨터의 여러 CPU）

```bash
#!/bin/bash
#SBATCH --job-name=mpi_single
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -N 1                   # 노드 1개
#SBATCH --ntasks=40            # MPI 프로세스 40개
#SBATCH --cpus-per-task=1      # 프로세스당 CPU 1개

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### 멀티 노드 MPI（여러 컴퓨터 협력 계산） — 가장 중요！

```bash
#!/bin/bash
#SBATCH --job-name=mpi_multi
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p cpu
#SBATCH -N 3                   # 3개 노드 사용
#SBATCH --ntasks=12            # 총 MPI 프로세스 12개
#SBATCH --ntasks-per-node=4    # 노드당 4개 프로세스
#SBATCH --cpus-per-task=1

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### GPU + MPI 멀티 노드

```bash
#!/bin/bash
#SBATCH --job-name=gpu_mpi
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p gpu
#SBATCH -N 2                   # gnode 2대
#SBATCH --ntasks=8             # 총 MPI 프로세스 8개
#SBATCH --ntasks-per-node=4    # 노드당 4개（GPU 4개에 대응）
#SBATCH --gres=gpu:4           # 노드당 GPU 4개
#SBATCH --cpus-per-task=4

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### MPI + OpenMP 하이브리드 병렬

멀티 노드（MPI）와 멀티 스레드（OpenMP）를 동시에 사용하는 프로그램：

```bash
#!/bin/bash
#SBATCH --job-name=hybrid
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH -p cpu
#SBATCH -N 3                   # 3개 노드
#SBATCH --ntasks=6             # 총 MPI 프로세스 6개（노드당 2개）
#SBATCH --ntasks-per-node=2
#SBATCH --cpus-per-task=20     # MPI 프로세스당 OpenMP 스레드 20개

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

cd $SLURM_SUBMIT_DIR
srun ./myprogram
```

---

### 스크립트 안에서 MPI 환경 확인

```bash
#!/bin/bash
#SBATCH -p cpu
#SBATCH -N 2
#SBATCH --ntasks=8

srun bash -c '
echo "Rank $SLURM_PROCID / Total $SLURM_NTASKS — $(hostname) 에서 실행 중"
'
```

출력 예시：
```
Rank 0 / Total 8 — cnode1 에서 실행 중
Rank 1 / Total 8 — cnode1 에서 실행 중
Rank 4 / Total 8 — cnode2 에서 실행 중
...
```

---

### MPI 관련 환경 변수

| 변수 | 설명 |
|------|------|
| `$SLURM_NTASKS` | 총 MPI 프로세스 수 |
| `$SLURM_PROCID` | 현재 프로세스의 Rank（0 부터 시작）|
| `$SLURM_NODELIST` | 작업에 참여한 전체 노드 목록 |
| `$SLURM_NNODES` | 총 노드 수 |
| `$SLURM_NTASKS_PER_NODE` | 노드당 프로세스 수 |
| `$SLURM_CPUS_PER_TASK` | 프로세스당 CPU 수（OpenMP 용）|

---

### MPI 트러블슈팅

**Q: 노드 간 작업에서 "srun 을 찾을 수 없음" 오류가 발생함**  
A: 수정 완료（2026-05-25）. 전체 계산 노드에 `/usr/bin/srun` 설치 완료.

**Q: MPI 프로세스가 모두 같은 노드에서 실행됨**  
A: `-N 노드수` 와 `--ntasks` 를 둘 다 올바르게 지정해 주세요.

**Q: GPU MPI 작업에서 GPU 가 보이지 않음**  
A: `-p gpu` 와 `--gres=gpu:N` 둘 다 필수입니다. 하나라도 빠지면 안 됩니다.

**Q: 꼭 mpirun 을 써야 하는 경우**  
```bash
mpirun --mca btl_tcp_if_include eth0 -np 8 ./myprogram
# 하지만 srun 을 강력 권장합니다
```

---

## 자주 하는 실수

### ❌ GPU 작업에서 `-p gpu` 를 잊어버림
```bash
# 잘못됨 — CPU 노드에 투입되어 GPU 를 사용할 수 없음
sbatch --gres=gpu:2 job.sh

# 올바름
sbatch -p gpu --gres=gpu:2 job.sh
```

### ❌ 멀티 노드 MPI 에서 --ntasks 를 잊어버림
```bash
# 잘못됨 — 노드당 1개 프로세스만 실행됨
sbatch -N 3 job.sh

# 올바름 — 총 프로세스 수를 명시
sbatch -N 3 --ntasks=12 job.sh
```

---

*최종 업데이트: 2026-05-25*
