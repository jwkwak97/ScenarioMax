# 서버 환경 설치 및 실행 가이드

> 이 문서는 **NVIDIA B200 GPU 서버** (Ubuntu, JupyterHub) 환경에서  
> ScenarioMax를 이용해 **nuPlan DB → Waymax TFRecord** 변환을 실행하는 방법을 정리합니다.  
> 일반 환경에서의 설치는 [README.md](../README.md)를 참조하세요.

---

## 서버 환경 사양

| 항목 | 값 |
|------|-----|
| GPU | NVIDIA B200 × 8 |
| RAM | 2.2 TB |
| OS | Linux (Ubuntu) |
| Python | 3.10 (conda) |
| 패키지 관리 | conda + uv |

---

## 1. 설치

### 1-1. conda 환경 생성

```bash
conda create -n scenariomax python=3.10 -y
conda activate scenariomax
```

### 1-2. uv 설치

```bash
pip install uv
```

### 1-3. ScenarioMax 클론 및 설치

```bash
cd /home/jovyan/workspace
git clone https://github.com/jwkwak97/ScenarioMax.git
cd ScenarioMax

# nuPlan 지원 포함 설치 (uv 사용)
uv pip install -e ".[nuplan]"
```

### 1-4. 의존성 충돌 해결 (서버 전용)

이 서버 환경에서는 다음 문제가 발생합니다. 설치 후 반드시 수행하세요.

#### 문제 1: OpenCV libGL 오류
```
ImportError: libGL.so.1: cannot open shared object file: No such file or directory
```
**해결**:
```bash
pip uninstall opencv-python -y
pip install opencv-python-headless "numpy<2"
```

#### 문제 2: numpy 2.x 충돌
headless OpenCV 설치 시 numpy가 2.x로 업그레이드될 수 있음.
```bash
pip install "numpy<2"
```

#### 문제 3: libstdc++ CXXABI 버전 오류 (런타임)
```
ImportError: /usr/lib/x86_64-linux-gnu/libstdc++.so.6: version `CXXABI_1.3.15' not found
```
**해결**: 실행 시 `LD_PRELOAD` 환경변수로 우회 (아래 실행 명령어 참조).

### 1-5. 설치 확인

```bash
/home/jovyan/.conda/envs/scenariomax/bin/scenariomax-convert --help
```

---

## 2. 데이터 경로

> **주의**: 마운트된 볼륨 이름은 서버 환경마다 다를 수 있습니다 (예: `aitc-plan-team-1`, `aitc-plan-vmax-datavol-1` 등).  
> 아래와 같이 `NUPLAN_DATASETS_ROOT`를 먼저 설정하면 이후 모든 명령어를 그대로 사용할 수 있습니다.

```bash
# 본인 환경에 맞게 수정
export NUPLAN_DATASETS_ROOT=/home/jovyan/<마운트된-볼륨-이름>
```

| 항목 | 경로 |
|------|------|
| nuPlan DB | `${NUPLAN_DATASETS_ROOT}/data/cache/` |
| nuPlan Maps | `${NUPLAN_DATASETS_ROOT}/nuplan-maps-v1.0/` |
| 출력 위치 | `/home/jovyan/workspace/vmax_data/nuplan_tfrecord/` |

nuPlan DB 폴더 구조:
```
${NUPLAN_DATASETS_ROOT}/data/cache/
├── train_boston/      # 학습 데이터 (Boston)
├── train_pittsburgh/  # 학습 데이터 (Pittsburgh)
├── train_singapore/   # 학습 데이터 (Singapore)
└── val/               # 검증 데이터
```

---

## 3. 변환 실행

> **필수**: 실행 시 `LD_PRELOAD`, `NUPLAN_MAPS_ROOT`, `NUPLAN_DATA_ROOT` 환경변수를 반드시 설정해야 합니다.

### 테스트 변환 (소규모 확인용)

```bash
LD_PRELOAD=/home/jovyan/.conda/envs/vmax/lib/libstdc++.so.6 \
NUPLAN_MAPS_ROOT=${NUPLAN_DATASETS_ROOT}/nuplan-maps-v1.0 \
NUPLAN_DATA_ROOT=${NUPLAN_DATASETS_ROOT}/data \
/home/jovyan/.conda/envs/scenariomax/bin/scenariomax-convert \
  --nuplan_src ${NUPLAN_DATASETS_ROOT}/data/cache/train_boston \
  --dst /home/jovyan/workspace/vmax_data/scenariomax_test \
  --target_format tfexample \
  --num_workers 4 \
  --num_files 10
```

- `--num_files 10`: DB 파일 10개만 처리 (테스트용)
- 소요 시간: 약 5~6분
- 출력: `<dst>/training.tfrecord` (약 19MB)

### 전체 변환 (train_boston)

```bash
LD_PRELOAD=/home/jovyan/.conda/envs/vmax/lib/libstdc++.so.6 \
NUPLAN_MAPS_ROOT=${NUPLAN_DATASETS_ROOT}/nuplan-maps-v1.0 \
NUPLAN_DATA_ROOT=${NUPLAN_DATASETS_ROOT}/data \
/home/jovyan/.conda/envs/scenariomax/bin/scenariomax-convert \
  --nuplan_src ${NUPLAN_DATASETS_ROOT}/data/cache/train_boston \
  --dst /home/jovyan/workspace/vmax_data/nuplan_tfrecord/train_boston \
  --target_format tfexample \
  --num_workers 8
```

- `--num_files` 인수 없음 → 전체 DB 파일 처리
- `--num_workers 8`: 병렬 워커 수 (서버 CPU 288코어 → 최대 활용 가능)

### 장기 실행 시 tmux 사용 (SSH 끊겨도 유지)

```bash
# 새 tmux 세션 시작
tmux new -s scenariomax_convert

# 세션 안에서 변환 실행
LD_PRELOAD=/home/jovyan/.conda/envs/vmax/lib/libstdc++.so.6 \
NUPLAN_MAPS_ROOT=${NUPLAN_DATASETS_ROOT}/nuplan-maps-v1.0 \
NUPLAN_DATA_ROOT=${NUPLAN_DATASETS_ROOT}/data \
/home/jovyan/.conda/envs/scenariomax/bin/scenariomax-convert \
  --nuplan_src ${NUPLAN_DATASETS_ROOT}/data/cache/train_boston \
  --dst /home/jovyan/workspace/vmax_data/nuplan_tfrecord/train_boston \
  --target_format tfexample \
  --num_workers 8

# 세션 분리 (변환 유지): Ctrl+B → D
# 나중에 재접속: tmux attach -t scenariomax_convert
```

---

## 4. 출력 파일 구조

```
<dst>/
└── training.tfrecord    ← V-Max에서 직접 사용 가능한 TFRecord
```

### V-Max에서 사용하기

```bash
cd /home/jovyan/workspace/V-Max

/home/jovyan/.conda/envs/vmax/bin/python vmax/scripts/training/train.py \
  algorithm=td3_trajectory \
  "network/encoder=wayformer" \
  path_dataset=/home/jovyan/workspace/vmax_data/nuplan_tfrecord/train_boston/training.tfrecord \
  use_wandb=false
```

---

## 5. 주요 인수 정리

| 인수 | 설명 | 예시 |
|------|------|------|
| `--nuplan_src` | nuPlan DB 폴더 경로 | `.../data/cache/train_boston` |
| `--dst` | 출력 폴더 경로 | `.../vmax_data/train_boston` |
| `--target_format` | 출력 형식 | `tfexample` (V-Max/Waymax 호환) |
| `--num_workers` | 병렬 워커 수 | `8` |
| `--num_files` | 처리할 DB 파일 수 (생략 시 전체) | `10` (테스트용) |

---

## 6. 문제 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| `CXXABI_1.3.15 not found` | 시스템 libstdc++ 버전 부족 | `LD_PRELOAD=/home/jovyan/.conda/envs/vmax/lib/libstdc++.so.6` 추가 |
| `libGL.so.1 not found` | GUI 버전 OpenCV 사용 | `pip install opencv-python-headless` |
| `numpy 2.x incompatible` | numpy 버전 자동 업그레이드 | `pip install "numpy<2"` |
| `UserWarning: _self_ missing` | hydra 1.1 경고 | 무시해도 됨 (동작에 영향 없음) |

---

## 7. 참고

- [ScenarioMax 공식 README](../README.md)
- [V-Max 프로젝트](https://github.com/jwkwak97/V-Max)
- [Waymax](https://github.com/waymo-research/waymax)
