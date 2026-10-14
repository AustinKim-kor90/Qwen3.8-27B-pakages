# Qwen3.8-27B 폐쇄망 설치 패키지

Qwen3.8-27B (Qwen3_5ForConditionalGeneration, Qwen3.6과 동일 아키텍처)

vLLM 0.31.0 · torch 2.13.0+cu130 · transformers 5.17.0 · Python 3.12 · Linux x86_64 (glibc 2.34 이상)

`parts/` 의 조각(95MB × 40개)을 합치면 `vllm-offline/` 폴더(wheel 200개 + 설치/실행 스크립트)가 복원됩니다.

## 폐쇄망 서버에서

```bash
git clone https://github.com/AustinKim-kor90/Qwen3.8-27B-pakages.git
cd Qwen3.8-27B-pakages
bash join.sh                       # 체크섬 확인 후 vllm-offline/ 복원
cd vllm-offline && bash install.sh # /opt/vllm 에 오프라인 설치 (VENV=경로 로 변경 가능)

MODEL_PATH=/data/models/Qwen3.8-27B TP=2 bash run.sh
```

- 기존 Gemma/EXAONE 환경과 별도 가상환경으로 설치되므로 기존 모델에 영향 없음
- OOM 시 `MAX_LEN=65536 bash run.sh`, 텍스트 전용이면 `bash run.sh --language-model-only`

자세한 설치 방법(다른 PC에서 받아 서버로 옮기는 경우 포함): [INSTALL_GUIDE.md](INSTALL_GUIDE.md)
