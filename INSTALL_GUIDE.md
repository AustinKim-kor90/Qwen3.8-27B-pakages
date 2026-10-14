# Qwen3.8-27B 폐쇄망 설치 가이드

다른 PC에서 clone하고 합친 뒤 **파일만 서버로 옮겨** 설치하는 방법입니다.

## 0. 서버 조건 확인

| 항목 | 조건 | 확인 명령 |
|---|---|---|
| OS | Linux x86_64 | `uname -m` → `x86_64` |
| glibc | 2.34 이상 (Ubuntu 22.04+, Ubuntu 24.04 권장) | `ldd --version` |
| Python | **3.12** | `python3.12 --version` |
| NVIDIA 드라이버 | CUDA 13.0 이상 지원 | `nvidia-smi` 오른쪽 위 `CUDA Version` |
| venv 모듈 | 설치되어 있어야 함 | `python3.12 -m venv /tmp/venvtest && echo OK` |

- CUDA Toolkit은 따로 설치할 필요 없음 (pip 패키지에 포함)
- 디스크 여유: wheel 3.7GB + 설치 약 7GB

## 1. 다른 PC에서 받기 + 합치기

GitHub에 접속 가능한 PC에서 진행합니다.

**Linux / macOS**
```bash
git clone https://github.com/AustinKim-kor90/Qwen3.8-27B-pakages.git
cd Qwen3.8-27B-pakages
bash join.sh            # 체크섬 확인 후 vllm-offline/ 폴더 생성
```

**Windows** — Git 설치 시 함께 깔리는 **Git Bash**를 열고 위와 같은 명령을 실행합니다. (PowerShell/CMD에서는 `join.sh`가 실행되지 않음)

`join.sh`가 `OK` 후 `완료`를 출력하면 성공입니다. 체크섬 불일치가 나오면 clone이 덜 된 것이니 다시 받으세요.

## 2. 서버로 옮길 파일

`vllm-offline/` 폴더 **하나만** 옮깁니다 (약 3.7GB).

```
vllm-offline/
├── wheels/            ← 패키지 200개
├── requirements.txt   ← 설치 목록 (필수)
├── install.sh
└── run.sh
```

- `.git/` 폴더와 `parts/` 폴더는 옮기지 마세요 (각 3.7GB, 설치에 불필요)
- 다른 PC에서 만든 **가상환경 폴더를 복사하는 것은 안 됩니다** (내부에 경로가 고정되어 깨짐). 설치는 반드시 서버에서.

```bash
# 예: scp 로 옮길 때
scp -r vllm-offline wyhil@서버주소:/home/wyhil/
```

root 등 다른 계정으로 복사했다면 권한 정리:
```bash
sudo chown -R wyhil:wyhil /home/wyhil/vllm-offline
```

> 서버로 조각 상태로 옮기고 싶다면 `parts/` 폴더와 `join.sh`만 옮긴 뒤 서버에서 `bash join.sh` 를 실행해도 됩니다. 이 경우 복사 중 손상도 체크섬으로 확인됩니다.

## 3. 서버에서 설치

### 방법 A — 이미 만들어 둔 가상환경에 설치 (현재 구성)

모델 폴더: `/home/wyhil/models/qwen3.8` (모델 파일 · 가상환경 · 실행 스크립트가 같은 폴더)

아래 `venv` 는 실제 가상환경 폴더 이름으로 바꾸세요.

```bash
# (1) 가상환경 파이썬 버전 확인 — 반드시 3.12
/home/wyhil/models/qwen3.8/venv/bin/python --version

# (2) 오프라인 설치
/home/wyhil/models/qwen3.8/venv/bin/pip install --no-index \
  --find-links /home/wyhil/vllm-offline/wheels \
  -r /home/wyhil/vllm-offline/requirements.txt

# (3) 확인 → "0.31.0 True" 가 나오면 성공
/home/wyhil/models/qwen3.8/venv/bin/python -c "import vllm, torch; print(vllm.__version__, torch.cuda.is_available())"
```

(1)에서 3.12가 아니면, 같은 이름으로 다시 만든 뒤 (2)부터 진행:
```bash
rm -rf /home/wyhil/models/qwen3.8/venv
python3.12 -m venv /home/wyhil/models/qwen3.8/venv
```

### 방법 B — 새 가상환경을 만들면서 설치

```bash
cd /home/wyhil/vllm-offline
VENV=/home/wyhil/models/qwen3.8/venv bash install.sh
```

- `VENV` 를 지정하지 않으면 `/opt/vllm` 에 설치되며 sudo 권한이 필요합니다.
- 가상환경은 **만든 뒤 경로를 옮기면 깨집니다.** 처음부터 계속 쓸 위치에 만드세요.

## 4. 실행

기존 실행 스크립트를 쓴다면 아래 옵션이 들어 있는지 확인하세요.

| 옵션 | 이유 |
|---|---|
| `HF_HUB_OFFLINE=1` (환경변수) | 폐쇄망에서 HF Hub 접속 시도로 멈추는 것 방지 |
| 모델 경로는 **로컬 폴더 경로** | `Qwen/Qwen3.8-27B` 처럼 HF ID를 쓰면 다운로드를 시도함 |
| `--reasoning-parser qwen3` | 없으면 `<think>` 내용이 답변에 섞여 나옴 |
| `--enable-auto-tool-choice --tool-call-parser qwen3_coder` | 툴 호출을 쓸 때 |
| `--max-model-len 131072` | 기본 262K는 OOM 나기 쉬움. 그래도 OOM이면 65536 |
| `--tensor-parallel-size N` | 80GB GPU 1장=1, 48GB급이면 2 이상 (BF16 가중치만 약 54GB) |
| `--language-model-only` | 이미지 입력을 안 쓰면 메모리 절약 |

예시:
```bash
source /home/wyhil/models/qwen3.8/venv/bin/activate
export HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 VLLM_NO_USAGE_STATS=1

vllm serve /home/wyhil/models/qwen3.8 \
  --served-model-name Qwen3.8-27B \
  --host 0.0.0.0 --port 8000 \
  --tensor-parallel-size 2 \
  --max-model-len 131072 \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice --tool-call-parser qwen3_coder
```

모델 폴더 안에 가상환경·스크립트가 같이 있어도 괜찮습니다. vLLM은 모델 폴더 최상위의 `*.safetensors` 만 읽고 하위 폴더는 보지 않습니다.

첫 기동은 커널 컴파일 때문에 몇 분 걸릴 수 있습니다 (정상).

## 5. 정리

설치가 끝나면 서버의 `/home/wyhil/vllm-offline` 은 지워도 됩니다.

## 문제 해결

| 증상 | 원인 / 조치 |
|---|---|
| `ensurepip is not available` | 서버에 `python3.12-venv` 없음 → 관리자에게 설치 요청 |
| `No matching distribution found` | 가상환경 파이썬이 3.12가 아님, 또는 `--find-links` 경로 오타 |
| `torch.cuda.is_available()` 가 `False` | NVIDIA 드라이버가 CUDA 13.0 미만 → 드라이버 업데이트 필요 |
| `GLIBC_2.34 not found` | OS가 너무 오래됨 (Ubuntu 22.04 미만) |
| 기동 중 OOM | `--max-model-len` 줄이기, `--language-model-only`, TP 늘리기 |
| HF Hub 접속 시도하며 멈춤 | 모델 경로를 로컬 폴더로, `HF_HUB_OFFLINE=1` 설정 |

## 버전

vLLM 0.31.0 · torch 2.13.0+cu130 · transformers 5.17.0 · Python 3.12 · Linux x86_64
