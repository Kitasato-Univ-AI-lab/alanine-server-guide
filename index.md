[English version](/alanine-server-guide/en/)

# alanine サーバー: SSH接続とSGEジョブ投入ガイド

## 1. SSH接続

### 1.1 接続先

- ホスト名: `alanine`
- グローバルIP: `202.251.201.250`
- ローカルIP: `192.168.0.6`
- SSHポート: `22` と `7522` の両方で待受
- SSH認証: 公開鍵認証のみ (`PasswordAuthentication no`)
- rootログイン: 禁止 (`PermitRootLogin no`)

### 1.2 事前準備（クライアントPC）

1. 鍵を持っていない場合は作成

```bash
ssh-keygen -t ed25519 -C "your_name@your_pc"
```

2. 公開鍵を管理者に渡し、サーバー側 `~/.ssh/authorized_keys` に登録してもらう

公開鍵の中身確認:

```bash
cat ~/.ssh/id_ed25519.pub
```

### 1.3 最小接続コマンド

```bash
ssh -p 22 <ユーザー名>@202.251.201.250
```

または:

```bash
ssh -p 7522 <ユーザー名>@202.251.201.250
```

### 1.4 `~/.ssh/config` 推奨設定

```sshconfig
Host alanine
  HostName 202.251.201.250
  User <ユーザー名>
  Port 22
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  ServerAliveInterval 30
  ServerAliveCountMax 3
```

設定後:

```bash
ssh alanine
```

### 1.5 初回接続時の確認

```bash
hostname
whoami
pwd
```

期待値:

- `hostname` が `alanine`
- `whoami` が自分のユーザー名

### 1.6 接続トラブル時の確認順

1. まず詳細ログで原因を見る

```bash
ssh -vvv -p 22 <ユーザー名>@202.251.201.250
```

2. `Permission denied (publickey)` の場合

- 公開鍵が `authorized_keys` に入っているか
- `~/.ssh` が `700`、`authorized_keys` が `600` か
- `ssh-add -l` で鍵が使われているか

3. タイムアウトする場合

- `22` が閉じているなら `7522` を試す
- 学内/自宅ネットワークのFW制限を確認

## 2. ジョブスケジューラ

この環境のジョブスケジューラは **SGE (Sun Grid Engine 8.1.9)** です。

主要コマンド:

- 投入: `qsub`
- 状態確認: `qstat`
- 削除: `qdel`
- ホスト確認: `qhost`
- 設定確認: `qconf`

## 3. キュー構成（実測）

現在確認できるキュー:

- `main.q` (CPU向け)
- `gpu.q` (GPU向け)
- `login.q` (対話用、INTERACTIVE)

並列環境(PE):

- `smp`
- `mpi`

ホスト例:

- `main.q`: `hpc52 hpc53 hpc54 hpc55 hpc56 hpc57`
- `gpu.q`: `hpc50 hpc51 hpc58`

確認コマンド:

```bash
qconf -sql
qstat -f
qhost
```

## 4. GPUサーバースペック（実測）

### 4.1 `qsub` 対象GPUノード

| ホスト | CPUスレッド (`nproc`) | メモリ | GPU |
|---|---:|---:|---|
| `hpc51` | 40 | 約503 GiB | Tesla P100-SXM2-16GB x8 |
| `hpc58` | 96 | 約1.0 TiB | A100-PCIE-40GB x8 |
| `hpc50` | 40 | 約256 GiB (`qconf`定義) | `gpu=3` (`qconf`定義) |

補足:

- `hpc50` は確認時にネットワーク到達不可だったため、`qconf` の設定値を記載
- 現在の `gpu.q` は `hpc50 hpc51 hpc58` で構成

### 4.2 `qsub` 外のGPUサーバー

| ホスト | CPUスレッド (`nproc`) | メモリ | GPU |
|---|---:|---:|---|
| `hpch1` | 32 | 約1.0 TiB | NVIDIA H100 NVL x4 |
| `hpca1` | 64 | 約125 GiB | NVIDIA A100 80GB PCIe x1 |

## 5. 最小のバッチジョブ投入（CPU）

### 5.1 スクリプト例

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

# 実行したい処理
python3 -V

sleep 10
echo "END=$(date)"
```

### 5.2 投入

```bash
qsub job_cpu.sh
```

### 5.3 監視

```bash
qstat -u $USER
qstat -j <JOB_ID>
```

## 6. GPUジョブ投入

この環境では `gpu` という consumable resource が定義されています。
GPUジョブでは `-q gpu.q` と `-l gpu=<必要枚数>` を指定してください。
実運用では `-l h=<node>` でノード固定し、`CUDA_VISIBLE_DEVICES` を合わせて指定する形もよく使われます。

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

投入:

```bash
qsub job_gpu.sh
```

### 6.1 ノード固定でGPUジョブを投げる例

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
conda activate <env名>
python Train.py
```

## 7. 対話実行

### 7.1 `qlogin`（対話シェル）

```bash
qlogin -q login.q
```

### 7.2 `qrsh`（単発コマンド）

```bash
qrsh -q main.q hostname
```

## 8. よく使う運用コマンド

### 8.1 自分のジョブ一覧

```bash
qstat -u $USER
```

### 8.2 全体キュー状況

```bash
qstat -f
```

### 8.3 ジョブ詳細（pending理由も確認可能）

```bash
qstat -j <JOB_ID>
```

### 8.4 ジョブ削除

```bash
qdel <JOB_ID>
```

### 8.5 投入前の適合性チェック

```bash
qsub -w v -q main.q -pe smp 4 -l h_vmem=4G -b y /bin/true
qsub -w v -q gpu.q  -pe smp 8 -l h_vmem=8G,gpu=1 -b y /bin/true
```

`verification: found suitable queue(s)` が出れば、要求条件は満たしています。

## 9. 配列ジョブ

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

投入:

```bash
qsub job_array.sh
```

## 10. 推奨運用ルール

- `-cwd` を常につける（相対パス事故を防ぐ）
- `-j y` と `-o` でログを明示
- メモリは `h_vmem` を明示する
- GPUジョブは `gpu.q` + `gpu` resource を必ず指定
- 長時間処理は必ず `tmux` ではなく `qsub` で実行する

## 11. 詰まりやすいポイント

### 11.1 ジョブが `qw` のまま

確認順:

1. `qstat -j <JOB_ID>` で `scheduling info` を確認
2. `qstat -f` で該当キューの状態 (`au` など) を確認
3. 資源要求が厳しすぎないか確認 (`h_vmem`, `gpu`, `-pe smp`)
4. `qsub -w v ...` で事前検証

### 11.2 GPUジョブが期待どおり動かない

確認順:

1. `qsub` 指定を確認 (`-q gpu.q`, `-l gpu=<n>`, `-l h_vmem=...`)
2. 実行ホストを固定したい場合は `-l h=<node>` を追加
3. ジョブ内で `nvidia-smi` と `torch.cuda.device_count()` を出力
4. `CUDA_VISIBLE_DEVICES` を使う場合は `gpu=<n>` と個数を合わせる

## 12. すぐ使えるテンプレート（コピペ用）

### 12.1 CPU 1コア最小

```bash
qsub -cwd -N quick_cpu -q main.q -pe smp 1 -l h_vmem=2G -b y '/bin/bash -lc "hostname; date"'
```

### 12.2 GPU 1枚最小

```bash
qsub -cwd -N quick_gpu -q gpu.q -pe smp 1 -l h_vmem=4G,gpu=1 -b y '/bin/bash -lc "hostname; nvidia-smi -L"'
```

### 12.3 実運用テンプレート（GPU学習）

```bash
#$ -S /bin/bash
#$ -cwd
#$ -j y
#$ -l h=hpc58
#$ -l h_vmem=200g
#$ -l gpu=2
export CUDA_VISIBLE_DEVICES="4,5"

source $HOME/.pyenv/versions/anaconda3-2023.03/bin/activate
conda activate <env名>
python Train.py
```

## 13. CUDA運用

### 13.1 CUDA確認コマンド（ジョブ内で実行）

```bash
which nvcc
nvcc --version
nvidia-smi
```

補足:

- `nvcc --version` は `Cuda compilation tools, release 10.1, V10.1.243` を返す
- `nvidia-smi` 側では Driver 460.73.01 / CUDA Version 11.2 と表示される
- 実行時は、使用するフレームワーク（PyTorch等）の対応CUDAバージョンを必ず確認する

### 13.2 ジョブスクリプト先頭の推奨形

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

### 13.3 Python（PyTorch等）利用時

- GPU可否チェックは `qsub` ジョブ内で実行する
- 依存ライブラリは `venv`/`conda` を使ってプロジェクトごとに固定する

例:

```bash
python - <<'PY'
import torch
print("torch:", torch.__version__)
print("cuda available:", torch.cuda.is_available())
print("device count:", torch.cuda.device_count())
PY
```

## 14. `hpch1` / `hpca1` の使い方（直接実行）

`hpch1` と `hpca1` は `qsub` 配信先ではないため、SSHでログインして実行する。

`alanine` にログイン済みなら、次で直接入れる:

```bash
ssh hpca1
```

### 14.1 接続と基本確認

```bash
ssh <ユーザー名>@hpch1
hostname
nvidia-smi -L
```

`hpca1` を使う場合:

```bash
ssh <ユーザー名>@hpca1
hostname
nvidia-smi -L
```

### 14.2 実行例（`tmux` 推奨）

```bash
tmux new -s train
source $HOME/.pyenv/versions/anaconda3-2023.03/bin/activate
conda activate <env名>
export CUDA_VISIBLE_DEVICES="0,1,2,3"
python Train.py
```

`hpca1`（GPU 1枚）では、例えば次のように実行する:

```bash
tmux new -s train
source $HOME/.pyenv/versions/anaconda3-2023.03/bin/activate
conda activate <env名>
export CUDA_VISIBLE_DEVICES="0"
python Train.py
```

デタッチ:

```bash
Ctrl-b d
```

再接続:

```bash
tmux attach -t train
```
