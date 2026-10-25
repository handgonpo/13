## 1. 오늘의 목표

7일차에는 우리가 만든 PC AI 플랫폼을 Docker Compose로 실행했습니다. `api`, `db`, `redis`, `worker` 네 컨테이너가 FastAPI·PostgreSQL·Redis·Celery를 담당했습니다. 그동안 실제 AI 추론 모델은 교과 9·10의 기존 실행환경이나 Raspberry Pi처럼 **별도 장치**에서 실행했습니다.

오늘은 교과 10에서 사용한 **`yolo11n-pose.pt`** 모델을 **NVIDIA Jetson Orin Nano Super의 GPU 컨테이너**에 배포합니다. Jetson에서 사람과 COCO 17 Keypoint를 실제로 추론한 뒤, 결과를 **7일차 PC FastAPI 플랫폼을 통해 2일차 PostgreSQL의 `inference_events` 테이블에 저장**합니다.

오늘 수업이 끝나면 다음 작업을 직접 수행할 수 있습니다.

- Jetson의 ARM64 아키텍처, JetPack, CUDA, NVIDIA Container Runtime을 확인합니다.
- PC의 일반 Docker 이미지와 Jetson GPU Docker 이미지가 다른 이유를 이해합니다.
- 5일차 `artifacts/day05/jetson/`에 준비한 실제 모델 가중치와 SHA256을 확인합니다.
- Jetson 전용 Dockerfile과 Compose 파일을 작성하고 GPU 추론 서버를 실행합니다.
- 사람이 나온 JPG/PNG 이미지에서 실제 Bounding Box, COCO 17 Keypoint와 검출 신뢰도를 얻습니다.
- PC FastAPI가 Jetson에 이미지를 보내고, 결과를 공통 `InferenceEvent`로 변환하여 PostgreSQL에 저장합니다.
- 웹브라우저에서 이미지 위에 Bounding Box·Skeleton을 표시하고 저장 이력을 다시 조회합니다.
- 인증 실패·모델 누락·GPU 연결 문제와 네트워크 연결 문제를 구분합니다.
- 자동 테스트·README·Git 기록으로 오늘 작업을 마무리합니다.

**오늘의 최종 완성 기준:** 기존 PC 네 컨테이너를 유지하면서 Jetson의 `pose` 컨테이너가 GPU로 이미지를 분석하고, 웹 화면에서 사람 Keypoint를 표시하며 PostgreSQL에 `jetson:` 이벤트가 남아 있어야 합니다.

---

## 2. 오늘 수업의 전체 흐름

![[Pasted image 20261018131344.png]]

```text
[1~7일차에 만든 기존 PC 플랫폼]
FastAPI(api) · PostgreSQL(db) · Redis(redis) · Celery(worker)
                              │
                              │ 기존 API/DB 그대로 유지
                              ↓
[5일차 준비 결과 확인]
yolo11n-pose.pt + yolo11n-pose.onnx + Pose Manifest
                              ↓
                 모델 SHA256 검증
                              ↓
[Jetson Orin Nano Super]
JetPack / ARM64 / CUDA / Docker / NVIDIA Runtime 확인
                              ↓
                 Jetson 전용 Dockerfile
                              ↓
               Jetson GPU 컨테이너 실행
                              ↓
              YOLO11n-Pose GPU 추론 테스트
                              ↓
                     실제 JPG / PNG
                              ↓
                  Person BBox + COCO17
                              │
                              │ HTTP + 장치 인증 키
                              ↓
[PC의 기존 FastAPI]
이미지 전달 → Pose JSON 검증 → 공통 InferenceEvent
                              ↓
[기존 PostgreSQL]
inference_events + camera_sources + ai_models
                              ↓
[PC 웹브라우저]
이미지 + Bounding Box + Skeleton + 기록 조회
                              ↓
                Pytest · README · Git
                              ↓
[9일차] 같은 Jetson 모델을 ONNX·TensorRT·연속 영상으로 확장
```

**오늘 바꾸지 않는 것:** 1~7일차의 기존 `app/schemas.py`, `app/models.py`, `app/db.py`, 7일차 PC `Dockerfile`, Raspberry Pi 배포 프로그램, 교과 10 Random Forest 가중치는 수정하지 않습니다.

---

## 3. 실습 전에 알아야 할 핵심

### 3.1. 7일차 Docker Compose와 오늘 Jetson Docker는 어디에서 실행되나요?

| 구분 | 7일차 PC | 오늘 8일차 Jetson |
|---|---|---|
| 장치 | Windows/WSL 또는 Ubuntu 개발 PC | NVIDIA Jetson Orin Nano Super |
| CPU 구조 | 주로 `x86_64` / `amd64` | `aarch64` / `arm64` |
| 실행 대상 | FastAPI·PostgreSQL·Redis·Celery | 실제 YOLO11n-Pose GPU 추론 |
| 실행 파일 | 기존 `compose.yaml` | 새 `deploy/day08_jetson/compose.jetson.yaml` |
| Dockerfile | 기존 루트 `Dockerfile` | 새 `deploy/day08_jetson/Dockerfile.jetson` |
| 저장소 | 기존 PC PostgreSQL | Jetson에는 별도 PostgreSQL을 만들지 않음 |

PC에서 작성한 소스는 Jetson에 전송하지만 **PC의 `python:3.12-slim-bookworm` 이미지를 그대로 Jetson GPU용으로 실행하지 않습니다.** Jetson에는 JetPack 버전에 맞는 ARM64·CUDA·PyTorch 환경이 필요합니다.

### 3.2. 오늘 사용하는 AI 모델은 무엇인가요?

교과 10에서는 YOLO11n-Pose로 사람을 찾고 17개 Keypoint를 추출했습니다. 이 Keypoint에서 38-D Feature와 Sliding Window를 만든 뒤 Random Forest로 6개 Action을 분류했습니다.

오늘 Jetson 실습은 그중 **Pose 추론 단계**를 먼저 GPU에 배포합니다.

```text
오늘(8일차): 이미지 → YOLO11n-Pose GPU → Person BBox + 17 Keypoint

아직 수행하지 않음: 17 Keypoint → 38-D Feature → Sliding Window
                → Random Forest → 6개 Action 분류

다음(9일차): 동일 Pose 모델 ONNX → TensorRT · 카메라 연속 입력 · 성능 비교
```

따라서 오늘 결과의 `bbox_confidence`는 **사람 검출 신뢰도**이지 6개 동작에 대한 Random Forest 신뢰도가 아닙니다. 아직 업무 규칙에 의한 OK/NG를 결정하지 않았으므로 PC 공통 이벤트의 `decision`은 **`UNKNOWN`**, `reasons`는 `POSE_ONLY_NO_ACTION_DECISION`으로 기록합니다. 사람을 찾지 못한 경우에는 `NO_PERSON_DETECTED`로 구분합니다. 이것은 모델 배포 실패나 산업안전 위험 판정이 아닙니다.

### 3.3. 5일차에서 만들어 둔 파일을 사용합니다

`subject13_ai_platform`의 아래 세 파일이 필요합니다.

```text
artifacts/day05/jetson/
├── yolo11n-pose.pt                 # 교과 10에서 학습·사용한 실제 Pose 가중치
├── yolo11n-pose.onnx               # 5일차에 내보낸 ONNX 후보(9일차 준비)
└── pose_export_manifest.json      # ONNX SHA256·입력 크기·OPSET 기록
```

오늘의 실제 추론은 `.pt`를 Jetson GPU에서 실행합니다. `.onnx` 파일은 **9일차 TensorRT로 연결할 자산**이며, 오늘 TensorRT 엔진을 생성하지 않습니다.

**이전 실습 결과가 없다면:** 5일차를 완료한 동료 또는 제공받은 **교과 10의 실제 `yolo11n-pose.pt`**, 5일차의 `yolo11n-pose.onnx`, `pose_export_manifest.json`을 해당 폴더에 배치합니다. 새 가중치를 임의로 만들거나 다른 모델을 같은 파일명으로 바꾸지 않습니다. 교과 10에서 직접 사용했던 사람 이미지 JPG/PNG **1장 이상**도 필요합니다. 원본 이미지를 외부 공개하지 마세요.

### 3.4. JetPack과 CUDA를 확인해야 하는 이유

Jetson의 JetPack에는 GPU 드라이버와 CUDA 등 실행환경이 함께 포함됩니다. Docker 컨테이너가 GPU를 쓰려면 Jetson 호스트의 NVIDIA Container Runtime과 컨테이너의 라이브러리 구성이 호환되어야 합니다.

**이 교재의 Docker 경로는 JetPack 6 계열을 기준**으로 작성했습니다. 설치된 JetPack이 다르다면 이미지만 무작정 바꾸지 말고 실제 장치 버전과 맞는 Ultralytics/NVIDIA GPU 컨테이너를 확인해야 합니다. 특히 JetPack 5·7 이미지나 일반 `latest-arm64` CPU 이미지는 오늘의 JetPack 6 GPU 실습에 그대로 사용하지 않습니다.

### 3.5. 서로 다른 컴퓨터의 `localhost`를 구분합니다

```text
PC 브라우저       → http://127.0.0.1:8000       (PC Compose FastAPI)
PC FastAPI       → http://JETSON_LAN_IP:8018   (Jetson Pose 컨테이너)
Jetson 컨테이너  → Jetson의 NVIDIA GPU         (모델 추론)
PC FastAPI       → db:5432                    (기존 Compose PostgreSQL)
```

Jetson의 `127.0.0.1`은 Jetson 자신입니다. PC에서 Jetson에 요청하려면 **강의실에서 실제로 확인한 Jetson LAN IPv4**를 사용합니다. PC와 Jetson은 서로 접속 가능한 내부망이어야 하고, Jetson 포트 `8018`은 **PC에서만 접근할 수 있도록** 네트워크 정책을 지킵니다.

### 3.6. 오늘 저장하는 데이터와 저장하지 않는 데이터

PC PostgreSQL에는 이벤트 ID, 이미지 파일명, 모델 SHA256, GPU 이름, 추론 시간, 사람 수, Bounding Box·Keypoint 좌표 같은 **추론 결과 메타데이터**를 저장합니다. 이미지 원본의 바이트는 데이터베이스에 보관하지 않습니다. 실제 사람의 자세 정보도 개인정보가 될 수 있으므로 동의받은 교육용 이미지만 사용합니다.

또한 `DAY08_JETSON_API_KEY`는 Jetson 서버 요청을 허용할 때 사용하는 장치 간 공유 키입니다. `.env`, `.env.jetson`과 원본 모델은 Git에 올리지 않습니다. **PC의 웹 화면은 기본적으로 7일차처럼 로컬 바인딩**하고 공개 인터넷에 열지 않습니다.

### 3.7. 오늘 시작 전에 준비할 것

| 준비 항목 | 확인 기준 |
|---|---|
| PC | 7일차 `docker compose ps`에 `api`, `db`, `redis`, `worker`가 실행 중 |
| 기존 프로젝트 | `~/ai_vision/subject13_ai_platform`과 `app/main.py` 존재 |
| 모델 | 5일차 PT/ONNX/Manifest 모두 준비됨 |
| 사람 이미지 | 교과 10의 촬영 이미지 JPG/PNG 1장 이상 |
| Jetson | JetPack 6, Docker Compose v2, NVIDIA Container Runtime 사용 가능 |
| 연결 | PC에서 Jetson의 LAN IP로 SSH/HTTP 접근 가능 |
| 저장소 | PC의 2일차 PostgreSQL에 `inference_events` 테이블 존재 |

---

## 4. 오늘 사용할 기술 스택

| 기술 | 오늘 사용하는 목적 |
|---|---|
| Jetson Orin Nano Super / JetPack 6 | ARM64 GPU 배포 대상과 호스트 CUDA 환경 |
| Docker Engine / Docker Compose | Jetson 전용 Pose 추론 컨테이너 실행 |
| NVIDIA Container Runtime | Jetson GPU를 컨테이너에 제공 |
| Ultralytics YOLO11n-Pose / PyTorch CUDA | 이미지에서 사람 Bounding Box와 COCO 17 Keypoint 실제 추론 |
| ONNX | 5일차 모델 후보 무결성 확인, 9일차 TensorRT 준비 |
| SHA256 | PC에서 준비한 모델과 Jetson 모델이 같은지 확인 |
| FastAPI / HTTPX | PC ↔ Jetson 이미지 요청·응답 |
| Pydantic | Pose 응답의 자료형·필수 정보 검증 |
| SQLAlchemy / PostgreSQL | 기존 `inference_events` 테이블에 GPU 추론 결과 저장 |
| HTML / JavaScript Canvas | 원본 이미지에 Bounding Box·Skeleton 표시 |
| Pytest / Git | 자동 점검과 버전 관리 |

**실습 기준:** Jetson 쪽은 JetPack 6·ARM64 장치를 사용하고, PC 쪽은 7일차에 구성한 Docker Compose를 사용합니다. 수업 시간에 Jetson 컨테이너 이미지를 다운로드할 수 없으면 미리 승인된 내부 이미지 저장소 또는 오프라인 이미지 반입이 필요합니다.

---

# 5. 처음부터 끝까지 따라 하는 상세 실습

## PART 1. 7일차 프로젝트와 Jetson 실행환경을 확인합니다

### 실습 1. 7일차 프로젝트를 그대로 엽니다

**PC WSL/Ubuntu 터미널**에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
pwd
docker compose ps
```

`db`, `redis`, `api`, `worker`가 실행 중이면 정상입니다. 아직 실행하지 않았다면 7일차 수업에서 수행한 DB 복원·이관이 끝난 상태인지 먼저 확인하고 다음을 실행합니다.

```bash
docker compose up -d
docker compose ps
```

기존 PostgreSQL의 데이터를 삭제하는 `docker compose down -v`는 사용하지 않습니다. 오늘은 기존 플랫폼을 초기화하는 날이 아닙니다.

현재 프로젝트 파일도 확인합니다.

```bash
ls -lh app/main.py app/db.py app/models.py app/repository.py \
  app/container_api.py compose.yaml Dockerfile
```

**왜 확인하나요?** 오늘 새로 추가할 Jetson API는 2일차 SQLAlchemy 모델과 7일차 Docker Compose에 연결됩니다. 위 파일이 없으면 새 프로젝트를 만들어 대신 진행하지 말고 앞선 일차의 결과부터 확인합니다.

### 실습 2. 5일차에서 준비한 Pose 모델을 확인합니다

**PC 터미널**에서 다음 명령을 입력합니다.

```bash
ls -lh artifacts/day05/jetson/yolo11n-pose.pt \
  artifacts/day05/jetson/yolo11n-pose.onnx \
  artifacts/day05/jetson/pose_export_manifest.json
```

파일을 찾을 수 없으면 5일차 `scripts/15_prepare_jetson_pose.py`에서 수행한 **교과 10 YOLO Pose → Jetson용 PT/ONNX 준비** 부분을 확인합니다. 실제 모델을 확보하지 않고 다음 단계로 진행하지 않습니다.

### 실습 3. Jetson의 운영체제·CPU·Docker를 확인합니다

**Jetson 본체 터미널** 또는 PC에서 SSH로 접속한 Jetson 터미널에서 실행합니다. 아래 `JETSON_USER`, `JETSON_IP`는 예시가 아니라 **본인의 값**으로 지정해야 합니다.

PC에서 접속하는 경우:

```bash
export JETSON_USER=student
export JETSON_IP=192.168.0.31
ssh "${JETSON_USER}@${JETSON_IP}"
```

이제부터 다음 명령은 **Jetson 터미널**에서 실행합니다.

```bash
uname -m
cat /etc/nv_tegra_release
cat /etc/os-release | head -6
command -v docker
docker --version
docker compose version
sudo docker info --format '{{json .Runtimes}}'
```

`uname -m`이 `aarch64`이고 NVIDIA Runtime 목록에 `nvidia`가 나타나는지 확인합니다. `/etc/nv_tegra_release`의 `R36` 계열이 JetPack 6에 대응하지만, **정확한 버전은 장치에 설치된 JetPack 패키지와 함께 확인**합니다.

```bash
dpkg-query -W nvidia-jetpack 2>/dev/null || true
command -v tegrastats
```

`docker: permission denied`가 나타난다면 Jetson에서 본인 계정의 Docker 실행 권한을 확인합니다. 기관 정책상 Docker 그룹 사용이 허용된 경우에만 관리자 승인을 받아 다음처럼 설정하고 다시 로그인합니다.

```bash
sudo usermod -aG docker "$USER"
newgrp docker
docker ps
```

Docker 그룹에는 강한 시스템 관리 권한이 있으므로 허용되지 않은 장치에서는 임의로 설정하지 않습니다.

Docker 또는 NVIDIA Runtime이 없다면 Jetson의 **설치된 JetPack에 맞는 Docker·NVIDIA Container Toolkit**부터 준비해야 합니다. NVIDIA에서 제공하는 Jetson Docker 설정 절차를 확인하고 장치 관리자 지침에 따라 설치하세요. 임의의 PC CUDA 드라이버를 Jetson에 덮어쓰지 않습니다.

NVIDIA Runtime 설정이 이미 설치되어 있고 GPU 연결만 누락된 경우 다음 명령으로 Docker Runtime 설정을 갱신할 수 있습니다.

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
sudo docker info --format '{{json .Runtimes}}'
```

`nvidia-ctk: command not found`라면 컨테이너 도구가 준비되지 않은 상태입니다. 패키지·JetPack을 먼저 확인하고 장치에 맞는 설치를 진행해야 합니다. `docker compose` 자체가 없으면 Compose v2가 제공되도록 Jetson에 설치해야 합니다.

### 실습 4. Jetson에서 PyTorch CUDA 컨테이너를 사전 확인합니다

오늘 사용할 Ultralytics의 JetPack 6 전용 이미지를 Jetson에서 내려받습니다.

```bash
sudo docker pull ultralytics/ultralytics:latest-jetson-jetpack6
```

이 이미지는 Jetson ARM64용이며, 7일차 PC 이미지 `python:3.12-slim-bookworm`과 다른 환경입니다. 이미지가 준비되면 실제 GPU를 사용할 수 있는지 확인합니다.

```bash
sudo docker run --rm --runtime=nvidia \
  ultralytics/ultralytics:latest-jetson-jetpack6 \
  python3 -c 'import platform, torch; print("ARCH:",platform.machine()); print("CUDA:",torch.cuda.is_available()); print("GPU:",torch.cuda.get_device_name(0) if torch.cuda.is_available() else "not available")'
```

`ARCH: aarch64`, `CUDA: True`와 GPU 이름이 출력되어야 합니다. `CUDA: False`라면 앞으로 작성할 FastAPI 코드로 문제를 감추지 않고 **JetPack·NVIDIA Container Runtime·이미지 버전 호환성**을 먼저 해결합니다. JetPack 6.x 사이에도 패키지 차이가 있으므로 수업 전에 실제 장비에서 위 명령을 반드시 실행해 보세요.

---

## PART 2. 5일차 모델을 Jetson 배포용으로 검사합니다

### 실습 5. 8일차 파일 위치를 준비합니다

다시 **PC 터미널**로 돌아옵니다. PC에 남겨 둔 원래 프로젝트에서 진행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
mkdir -p deploy/day08_jetson/samples docs scripts tests
```

오늘 새로 추가할 파일의 위치는 다음과 같습니다. 기존 파일은 모두 그대로 둡니다.

```text
subject13_ai_platform/
├── app/
│   ├── main.py                         [기존 파일 맨 아래 Router 2줄 추가]
│   ├── jetson_api.py                   [신규: PC→Jetson 요청 + 공통 JSON]
│   └── jetson_repo.py                  [신규: 기존 PostgreSQL 테이블에 저장]
├── scripts/
│   └── 19_check_day08_inputs.py       [신규: 실제 Pose 모델·SHA256 검사]
├── deploy/
│   └── day08_jetson/
│       ├── app_jetson.py              [신규: Jetson GPU 추론 서버]
│       ├── requirements.jetson.txt    [신규]
│       ├── Dockerfile.jetson          [신규: Jetson ARM64/GPU]
│       ├── compose.jetson.yaml        [신규: Jetson 전용 Compose]
│       ├── .env.jetson.example        [신규: 비밀 값 없이 올리는 예시]
│       └── samples/                   [테스트 이미지, Git 제외]
├── artifacts/day05/jetson/            [기존 PT/ONNX, Git 제외]
│   ├── yolo11n-pose.pt
│   ├── yolo11n-pose.onnx
│   ├── pose_export_manifest.json
│   └── day08_manifest.json            [실습 6에서 자동 생성]
├── web/
│   └── jetson.html                    [신규]
├── tests/
│   └── test_day08.py                  [신규]
├── docs/
│   └── day08_jetson_runbook.md        [신규]
├── compose.yaml                       [기존: api 환경변수만 추가]
├── .env                               [기존: PC의 Jetson 연결 주소·키만 추가]
└── README.md                          [기존: 8일차 부분 추가]
```

이제 다음 실습에서 코드를 순서대로 작성합니다.

### 실습 6. 모델 SHA256을 비교하는 코드를 작성합니다

**왜 필요한가요?** Jetson으로 모델을 복사하다가 다른 버전을 가져오거나 파일이 손상되면 정상적인 성능 비교가 어렵습니다. PT와 ONNX의 해시를 계산하고 5일차 ONNX Manifest와 비교합니다.

**의사코드**

```text
5일차 PT / ONNX / Manifest 존재 여부 확인
                 ↓
           PT SHA256 계산
                 ↓
          ONNX SHA256 계산
                 ↓
     5일차 ONNX Manifest 해시와 비교
                 ↓
         모두 일치 → day08_manifest.json 저장
                 ↓
            Jetson 전송 준비 완료
```

**파일: `scripts/19_check_day08_inputs.py`**

```python
"""5·7일차 산출물과 Jetson 이관 전 모델 무결성을 검사합니다."""
import hashlib
import json
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]
MODEL_DIR = ROOT / "artifacts" / "day05" / "jetson"


def file_hash(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as handle:
        for chunk in iter(lambda: handle.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def main():
    required = [
        ROOT / "compose.yaml", ROOT / "app" / "main.py",
        ROOT / "app" / "db.py", ROOT / "app" / "models.py",
        MODEL_DIR / "yolo11n-pose.pt", MODEL_DIR / "yolo11n-pose.onnx",
        MODEL_DIR / "pose_export_manifest.json",
    ]
    absent = [str(path.relative_to(ROOT)) for path in required if not path.is_file()]
    if absent:
        raise FileNotFoundError("누락 파일: " + ", ".join(absent) + "\n5일차의 Jetson Pose 준비 실습부터 확인하세요.")
    original = json.loads((MODEL_DIR / "pose_export_manifest.json").read_text(encoding="utf-8"))
    pt_hash = file_hash(MODEL_DIR / "yolo11n-pose.pt")
    onnx_hash = file_hash(MODEL_DIR / "yolo11n-pose.onnx")
    if onnx_hash != original.get("onnx_sha256"):
        raise RuntimeError("5일차 ONNX 모델이 Manifest와 다릅니다. 파일을 확인하세요.")
    manifest = {"model_id": "yolo11n-pose", "runtime_day08": "pytorch_cuda",
                "pt_file": "yolo11n-pose.pt", "pt_sha256": pt_hash,
                "onnx_file": "yolo11n-pose.onnx", "onnx_sha256": onnx_hash,
                "note": "9일차 TensorRT 변환은 Jetson 장치에서 별도로 수행합니다."}
    output = MODEL_DIR / "day08_manifest.json"
    output.write_text(json.dumps(manifest, ensure_ascii=False, indent=2), encoding="utf-8")
    print("[PASS] 5일차 모델 · ONNX SHA256 · 7일차 소스 확인")
    print("[PASS] 8일차 Jetson 배포 모델 Manifest 생성:", output)
    print("PT SHA256:", pt_hash)


if __name__ == "__main__":
    main()
```

파일을 저장한 후 **PC의 기존 `.venv`**로 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python scripts/19_check_day08_inputs.py
```

정상이라면 다음과 같은 항목을 확인할 수 있습니다.

```text
[PASS] 5일차 모델 · ONNX SHA256 · 7일차 소스 확인
[PASS] 8일차 Jetson 배포 모델 Manifest 생성: .../day08_manifest.json
PT SHA256: <실제로 계산된 64자리 해시>
```

`<실제로 계산된 64자리 해시>`는 예시 안내이며 실제 명령은 모델의 고유 해시를 출력합니다. 해시를 문서 예시 값으로 바꾸지 않습니다. ONNX Manifest와 실제 파일이 다르면 파일을 교체하기 전에 원본 출처를 확인합니다.

---

## PART 3. Jetson 전용 GPU Docker 서비스를 작성합니다

### 실습 7. Jetson 전용 Python 패키지 목록을 작성합니다

**파일: `deploy/day08_jetson/requirements.jetson.txt`**

이 파일은 Jetson 이미지에 FastAPI 서버 기능을 추가합니다. Jetson 전용 Ultralytics 이미지에는 PyTorch·YOLO가 포함되어 있으므로 오늘 목록에 일반 PC용 `torch`를 다시 설치하지 않습니다.

```text
# JetPack 6 / Ultralytics GPU 베이스 이미지에 이미 있는 torch/ultralytics는 다시 설치하지 않습니다.
fastapi>=0.115,<1
uvicorn>=0.30,<1
python-multipart>=0.0.9,<1
```

### 실습 8. Jetson GPU Dockerfile을 작성합니다

**무슨 코드인가요?** JetPack 6 전용 Ultralytics 이미지를 기반으로, 우리가 만든 Jetson 추론 서버 코드를 포함하는 새 이미지를 만듭니다. **루트의 7일차 `Dockerfile`은 변경하지 않습니다.**

**의사코드**

```text
JetPack 6용 ARM64·CUDA 이미지 선택
                ↓
      FastAPI 서버 패키지 설치
                ↓
         app_jetson.py 복사
                ↓
        Jetson API 포트 8018 준비
                ↓
        단일 Uvicorn Worker 실행
```

**파일: `deploy/day08_jetson/Dockerfile.jetson`**

```dockerfile
# JetPack 6 전용 ARM64 CUDA/PyTorch 기반 이미지. PC의 7일차 Dockerfile과 별개입니다.
FROM ultralytics/ultralytics:latest-jetson-jetpack6
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /jetson_app
COPY requirements.jetson.txt ./requirements.jetson.txt
RUN python3 -m pip install --no-cache-dir -r requirements.jetson.txt
COPY app_jetson.py ./app_jetson.py
EXPOSE 8018
CMD ["python3", "-m", "uvicorn", "app_jetson:app", "--host", "0.0.0.0", "--port", "8018", "--workers", "1"]
```

`--workers 1`은 한 모델을 여러 프로세스에서 중복으로 로딩하여 Jetson 메모리를 낭비하지 않도록 하기 위한 기본값입니다. 이번 수업은 동시 처리량 최적화가 목적이 아닙니다.

### 실습 9. Jetson 실제 Pose GPU 추론 코드를 작성합니다

**이번 코드는 무엇을 하나요?** FastAPI 서버가 시작되면 실제 `.pt` 가중치와 SHA256을 확인하고, **CUDA가 사용할 수 없으면 서버 시작 자체를 중단**합니다. 처음 한 장을 추론하여 GPU 동작을 확인한 뒤, JPG/PNG를 받으면 실제 YOLO11n-Pose 모델로 사람과 17개 Keypoint를 예측합니다.

**의사코드**

```text
[Jetson 서버 시작]
/models/yolo11n-pose.pt 존재 확인
            ↓
.env.jetson의 SHA256과 실제 파일 SHA256 비교
            ↓
PyTorch torch.cuda.is_available() == True 확인
            ↓
YOLO 모델 로딩 → GPU Warm-up 추론
            ↓
FastAPI /health = ready

[이미지 요청]
장치 공유 키 확인
            ↓
JPG/PNG · 파일 크기 · 해상도 검사
            ↓
OpenCV로 실제 이미지 읽기
            ↓
YOLO.predict(device=0) GPU 추론
            ↓
Person Bounding Box / COCO17 좌표·Confidence 추출
            ↓
추론 시간·모델 SHA256·GPU 정보와 함께 JSON 응답
```

**파일: `deploy/day08_jetson/app_jetson.py`**

```python
"""교과13 8일차: Jetson GPU에서 실제 YOLO11n-Pose 이미지 추론."""
from __future__ import annotations

import hashlib
import os
import secrets
from contextlib import asynccontextmanager
from datetime import datetime, timezone
from pathlib import Path
from time import perf_counter
from uuid import uuid4

import cv2
import numpy as np
import torch
from fastapi import FastAPI, File, Header, HTTPException, UploadFile
from ultralytics import YOLO

MODEL_PATH = Path(os.getenv("JETSON_MODEL_PATH", "/models/yolo11n-pose.pt"))
MAX_IMAGE_BYTES = 5 * 1024 * 1024
MODEL = None
MODEL_SHA256 = None
GPU_NAME = None


def sha256_file(path: Path) -> str:
    """대용량 파일도 메모리에 전부 읽지 않고 SHA256을 계산합니다."""
    digest = hashlib.sha256()
    with path.open("rb") as stream:
        for chunk in iter(lambda: stream.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def verify_key(key: str | None) -> None:
    configured = os.getenv("JETSON_API_KEY", "")
    if len(configured) < 24:
        raise HTTPException(503, "Jetson 인증 키 설정이 없습니다.")
    if not key or not secrets.compare_digest(key, configured):
        raise HTTPException(401, "인증되지 않은 Jetson 추론 요청입니다.")


def startup_model() -> None:
    global MODEL, MODEL_SHA256, GPU_NAME
    if not MODEL_PATH.is_file():
        raise RuntimeError(f"모델 파일이 없습니다: {MODEL_PATH}")
    expected = os.getenv("JETSON_MODEL_SHA256", "").strip().lower()
    actual = sha256_file(MODEL_PATH)
    if len(expected) != 64 or actual != expected:
        raise RuntimeError("Jetson .pt 모델 SHA256이 .env.jetson과 일치하지 않습니다.")
    if not torch.cuda.is_available():
        raise RuntimeError("Jetson CUDA를 사용할 수 없습니다. NVIDIA Container Runtime을 확인하세요.")
    GPU_NAME = torch.cuda.get_device_name(0)
    MODEL = YOLO(str(MODEL_PATH))
    # 한 장의 실제 이미지로 최초 CUDA 추론 성공까지 확인합니다.
    warmup = np.zeros((640, 640, 3), dtype=np.uint8)
    MODEL.predict(source=warmup, device=0, imgsz=640, conf=0.25, verbose=False)
    MODEL_SHA256 = actual


@asynccontextmanager
async def lifespan(app: FastAPI):
    startup_model()
    yield


app = FastAPI(title="교과13 Jetson Pose GPU", version="day08", lifespan=lifespan)


@app.get("/health")
def health():
    return {"status": "ready" if MODEL is not None else "not_ready",
            "gpu_available": bool(torch.cuda.is_available()),
            "gpu_name": GPU_NAME,
            "model_id": "yolo11n-pose",
            "runtime": "pytorch_cuda", "device": "jetson"}


@app.post("/api/v1/jetson/pose")
async def pose_image(image: UploadFile = File(...),
                     x_jetson_key: str | None = Header(default=None, alias="X-Jetson-Key")):
    verify_key(x_jetson_key)
    if image.content_type not in {"image/jpeg", "image/png"}:
        raise HTTPException(415, "JPG 또는 PNG 이미지만 사용합니다.")
    raw = await image.read(MAX_IMAGE_BYTES + 1)
    if len(raw) > MAX_IMAGE_BYTES:
        raise HTTPException(413, "이미지 크기는 5MB 이하로 제한합니다.")
    if not raw:
        raise HTTPException(422, "이미지가 비어 있습니다.")
    frame = cv2.imdecode(np.frombuffer(raw, dtype=np.uint8), cv2.IMREAD_COLOR)
    if frame is None:
        raise HTTPException(422, "읽을 수 없는 이미지입니다.")
    if frame.shape[0] * frame.shape[1] > 4096 * 4096:
        raise HTTPException(413, "이미지 해상도가 너무 큽니다.")
    if MODEL is None:
        raise HTTPException(503, "YOLO Pose가 아직 준비되지 않았습니다.")

    start = perf_counter()
    outputs = MODEL.predict(source=frame, device=0, imgsz=640, conf=0.25,
                            max_det=5, verbose=False)
    inference_ms = round((perf_counter() - start) * 1000, 2)
    result = outputs[0]
    persons = []
    boxes = result.boxes
    if boxes is not None and len(boxes):
        xyxy = boxes.xyxy.cpu().tolist()
        scores = boxes.conf.cpu().tolist()
        keypoint_xy = result.keypoints.xy.cpu().tolist() if result.keypoints is not None else []
        keypoint_conf = (result.keypoints.conf.cpu().tolist()
                         if result.keypoints is not None and result.keypoints.conf is not None else [])
        for index, (box, score) in enumerate(zip(xyxy, scores)):
            joints = []
            if index < len(keypoint_xy):
                for joint_index, coords in enumerate(keypoint_xy[index]):
                    confidence = (keypoint_conf[index][joint_index]
                                  if index < len(keypoint_conf) else None)
                    joints.append([round(float(coords[0]), 2), round(float(coords[1]), 2),
                                   round(float(confidence), 4) if confidence is not None else None])
            persons.append({"bbox_xyxy": [round(float(v), 2) for v in box],
                            "bbox_confidence": round(float(score), 4),
                            "keypoints": joints})
    return {"event_id": str(uuid4()),
            "recorded_at": datetime.now(timezone.utc).isoformat(),
            "input_ref": Path(image.filename or "uploaded.jpg").name,
            "width": int(frame.shape[1]), "height": int(frame.shape[0]),
            "num_persons": len(persons), "persons": persons,
            "inference_ms": inference_ms, "device_id": "jetson-orin-nano-super",
            "model_id": "yolo11n-pose", "model_sha256": MODEL_SHA256,
            "runtime": "pytorch_cuda", "gpu_name": GPU_NAME,
            "note": "사람 Pose 검출이며 6개 Action 인식 또는 위험행동 판정이 아닙니다."}
```

**코드 이해하기**

`torch.cuda.is_available()`은 PyTorch가 Jetson GPU를 사용할 수 있는지 확인합니다. `device=0`은 첫 CUDA 장치를 지정합니다. `MODEL = YOLO(...)`를 요청마다 새로 만들지 않고 서버 시작 때 한 번만 로딩합니다. 실제 모델을 찾지 못하거나 CUDA가 `False`면 **CPU로 몰래 전환하지 않습니다**.

응답의 `bbox_xyxy`는 원본 이미지 **픽셀 좌표**이며 `[x1, y1, x2, y2]` 형식입니다. `keypoints`는 COCO 17개 순서의 `[x, y, confidence]`로 반환합니다. Keypoint Confidence가 없는 경우에는 `null`을 남기며 임의로 1.0을 채우지 않습니다. `inference_ms`는 이미지 전송·DB 저장 시간을 포함하지 않는 **서버의 모델 추론 구간**을 중심으로 측정합니다.

입력 파일은 최대 **5MB**이며 과도하게 큰 해상도를 제한합니다. 카메라 연속 프레임 처리나 TensorRT 엔진은 오늘 작성하지 않습니다.

### 실습 10. Jetson Docker Compose 파일을 작성합니다

**왜 별도의 Compose 파일인가요?** 7일차의 PostgreSQL·Redis 등은 PC에서 계속 실행합니다. Jetson에는 **GPU Pose 컨테이너 하나**만 올리면 됩니다. `runtime: nvidia`는 Jetson의 NVIDIA 컨테이너 기능을 사용하라는 뜻입니다.

**파일: `deploy/day08_jetson/compose.jetson.yaml`**

```yaml
# Jetson 장치에서만 실행. PC 7일차 compose.yaml은 수정하지 않고 별도 파일을 사용합니다.
services:
  pose:
    image: subject13-jetson-pose:day08
    build:
      context: .
      dockerfile: Dockerfile.jetson
    runtime: nvidia
    restart: unless-stopped
    environment:
      JETSON_MODEL_PATH: /models/yolo11n-pose.pt
      JETSON_MODEL_SHA256: ${JETSON_MODEL_SHA256:?모델 파일의 실제 SHA256을 입력하세요}
      JETSON_API_KEY: ${JETSON_API_KEY:?24자 이상의 장치 인증 키를 입력하세요}
    ports:
      - "${JETSON_BIND_HOST:?Jetson LAN IP를 입력하세요}:8018:8018"
    volumes:
      - ../../artifacts/day05/jetson:/models:ro
```

`../../artifacts/day05/jetson`은 이 Compose 파일이 위치한 `deploy/day08_jetson/`에서 프로젝트 루트의 모델 폴더로 올라가는 상대경로입니다. 실제 `.pt`와 ONNX는 **읽기 전용(`:ro`) 볼륨**으로 전달하므로 Docker 이미지에 가중치를 넣지 않습니다.

`JETSON_BIND_HOST`는 이 장치의 **실제 LAN IP**로 입력합니다. 오늘은 외부 인터넷에 서비스를 노출하지 않습니다. 공유 키는 `.env.jetson`에서 가져오며 코드와 Git에는 넣지 않습니다.

### 실습 11. Jetson 실행 설정 예시를 작성합니다

**파일: `deploy/day08_jetson/.env.jetson.example`**

```dotenv
# 아래 IP/KEY/HASH는 반드시 본인의 Jetson 장치에 맞게 변경합니다.
JETSON_BIND_HOST=192.168.0.31
JETSON_API_KEY=REPLACE_WITH_A_RANDOM_SECRET_AT_LEAST_24_CHARACTERS
JETSON_MODEL_SHA256=REPLACE_WITH_SHA256_OF_YOLO11N_POSE_PT
```

이 파일은 **실제 비밀번호·모델 해시를 저장하는 파일이 아닙니다.** 다음 PART에서 Jetson에 소스·모델을 전송한 뒤 프로젝트 루트의 `.env.jetson`을 별도로 생성합니다.

---

## PART 4. Jetson에 소스·모델을 전송하고 실제 GPU 추론을 실행합니다

### 실습 12. 사람 이미지와 Jetson 배포 파일을 준비합니다

**PC 터미널**에서 교과 10의 기존 사람 이미지를 확인합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
ls -lh ~/ai_vision/subject10_pose_action/data/day03/images/arms_up_person_a.jpg
```

파일이 있다면 Jetson 테스트에 사용할 폴더에 복사합니다.

```bash
cp ~/ai_vision/subject10_pose_action/data/day03/images/arms_up_person_a.jpg \
  deploy/day08_jetson/samples/test_pose.jpg
ls -lh deploy/day08_jetson/samples/test_pose.jpg
```

본인의 실제 이미지 파일명이 다르면 위의 원본 경로만 바꾸세요. **사람이 실제로 나온 이미지**여야 합니다. 결석 등으로 이전 이미지가 없다면 사용 허가를 받은 교과 10의 실제 입력 이미지 1장을 제공받아 같은 경로에 배치합니다. 개인 사진·영상은 원격 공개 저장소에 업로드하지 않습니다.

### 실습 13. PC에서 Jetson으로 필요한 파일만 전송합니다

**PC 터미널**에서 Jetson LAN IP와 계정을 확인합니다. 아래 값은 사용 예시이므로 실제 주소·계정으로 변경합니다.

```bash
export JETSON_USER=student
export JETSON_IP=192.168.0.31
ssh "${JETSON_USER}@${JETSON_IP}" 'mkdir -p ~/ai_vision/subject13_ai_platform/deploy/day08_jetson ~/ai_vision/subject13_ai_platform/artifacts/day05/jetson'
```

`rsync`로 이번 8일차 소스와 모델을 복사합니다.

```bash
rsync -av --exclude='__pycache__/' \
  deploy/day08_jetson/ \
  "${JETSON_USER}@${JETSON_IP}:~/ai_vision/subject13_ai_platform/deploy/day08_jetson/"

rsync -av artifacts/day05/jetson/yolo11n-pose.pt \
  artifacts/day05/jetson/yolo11n-pose.onnx \
  artifacts/day05/jetson/pose_export_manifest.json \
  artifacts/day05/jetson/day08_manifest.json \
  "${JETSON_USER}@${JETSON_IP}:~/ai_vision/subject13_ai_platform/artifacts/day05/jetson/"
```

`rsync: command not found`라면 두 PC/Jetson에 `rsync`를 설치하거나 허가된 내부 파일 전송 방법을 사용합니다. 원본 모델은 그대로 두고 **복사**합니다.

### 실습 14. Jetson에서 전송된 모델의 SHA256을 확인합니다

**Jetson 터미널**에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
ls -lh artifacts/day05/jetson/
sha256sum artifacts/day05/jetson/yolo11n-pose.pt
python3 - <<'PY'
import hashlib, json
from pathlib import Path
base = Path('artifacts/day05/jetson')
manifest = json.loads((base/'day08_manifest.json').read_text(encoding='utf-8'))
for name, key in [('yolo11n-pose.pt','pt_sha256'), ('yolo11n-pose.onnx','onnx_sha256')]:
    actual = hashlib.sha256((base/name).read_bytes()).hexdigest()
    assert actual == manifest[key], f'{name}: 모델 해시가 다릅니다.'
    print('[PASS]', name, actual[:16]+'...')
PY
```

두 파일이 PASS여야 합니다. SHA256이 다르면 `rsync`부터 다시 확인하고, 모델을 임의로 바꾸지 않습니다.

### 실습 15. Jetson에서 `.env.jetson`을 작성합니다

실제 모델 파일 해시와 장치 IP를 알아야만 실행 설정을 만들 수 있습니다. **Jetson 터미널**에서 예시 파일을 복사합니다.

```bash
cp deploy/day08_jetson/.env.jetson.example .env.jetson
python3 -c 'import secrets; print(secrets.token_urlsafe(32))'
sha256sum artifacts/day05/jetson/yolo11n-pose.pt
hostname -I
nano .env.jetson
```

`nano`에서 세 값을 직접 입력합니다.

```dotenv
JETSON_BIND_HOST=본인_Jetson_LAN_IPv4
JETSON_API_KEY=방금_생성한_24자_이상_임의의_키
JETSON_MODEL_SHA256=방금_계산한_yolo11n-pose.pt의_SHA256_64자리
```

위의 한글 문자열은 **입력 항목을 설명하는 자리 표시자**이므로 실제 값으로 교체해야 합니다. `nano`는 `Ctrl+O` → Enter로 저장하고 `Ctrl+X`로 종료합니다.

```bash
chmod 600 .env.jetson
```

**중요:** 생성한 `JETSON_API_KEY`는 뒤의 PC `.env`에서 `DAY08_JETSON_API_KEY`에 **정확하게 같은 값**으로 입력합니다. 키를 README나 소스코드·Git에 붙여 넣지 마세요.

### 실습 16. Jetson 전용 Docker Compose 설정을 확인합니다

**Jetson 터미널**에서 실행합니다.

```bash
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml config --quiet
```

오류 없이 종료되면 Compose 구문과 필수 설정이 준비된 것입니다. `config` 전체를 화면에 출력하면 장치 인증 키가 노출될 수 있으므로 `--quiet`를 사용합니다.

### 실습 17. Jetson GPU 이미지·컨테이너를 빌드하고 실행합니다

**Jetson 터미널**에서 다음 명령을 순서대로 실행합니다.

```bash
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml build pose

docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml up -d pose

docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml ps
```

처음 빌드하는 경우 네트워크와 패키지 설치 때문에 시간이 걸릴 수 있습니다. 정상 실행 상태가 아니라면 로그를 확인합니다.

```bash
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml logs --tail=70 pose
```

**정상 기준:** 모델 파일 SHA256 확인과 GPU Warm-up을 통과하여 컨테이너가 실행 중이어야 합니다. GPU가 `False`일 때 자동으로 CPU를 사용하는 코드를 만들지 않았으므로 NVIDIA Runtime 문제가 있으면 로그에 오류가 남습니다.

### 실습 18. Jetson 컨테이너 안에서 실제 CUDA 상태를 확인합니다

**Jetson 터미널**에서 실행합니다.

```bash
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml exec pose \
  python3 -c 'import torch,platform; print("ARCH",platform.machine()); print("CUDA",torch.cuda.is_available()); print("GPU",torch.cuda.get_device_name(0) if torch.cuda.is_available() else "none")'
```

`ARCH aarch64`, `CUDA True`, 실제 Jetson GPU 이름이 출력되는지 확인합니다. `nvidia-smi`는 일부 Jetson에서 사용하지 않으므로 **CUDA 상태와 `tegrastats` 등 Jetson 장치 지표**를 기준으로 확인합니다.

### 실습 19. Jetson에서 이미지 한 장을 실제로 추론합니다

먼저 Jetson의 LAN IP로 API가 열리는지 확인합니다. `JETSON_HOST_IP`에는 현재 장치에서 접속 가능한 실제 IP를 입력합니다.

```bash
export JETSON_HOST_IP=192.168.0.31
curl --fail --max-time 10 "http://${JETSON_HOST_IP}:8018/health"
```

정상이라면 `status: ready`, `gpu_available: true`, `runtime: pytorch_cuda`가 JSON으로 반환됩니다.

이어서 **Jetson의 실제 모델로** 테스트 이미지를 분석합니다.

```bash
curl --fail --max-time 65 -X POST \
  "http://${JETSON_HOST_IP}:8018/api/v1/jetson/pose" \
  -H "X-Jetson-Key: $(sed -n 's/^JETSON_API_KEY=//p' .env.jetson)" \
  -F "image=@deploy/day08_jetson/samples/test_pose.jpg;type=image/jpeg" \
  | python3 -m json.tool
```

정상 응답에는 다음과 같은 필드가 나타납니다. **좌표, 시간, 사람 수는 실제 이미지·장비 실행 결과에 따라 달라집니다.**

```text
status: 성공 시 HTTP 200
recorded_at: 실제 UTC 추론 시각
event_id: 요청마다 생성된 UUID
width / height: 실제 입력 이미지 해상도(양수)
num_persons: 검출한 사람 수
persons[]: 사람별 bbox_xyxy / bbox_confidence / keypoints
inference_ms: 해당 장치의 추론 측정 시간
model_sha256: 배포한 실제 PT 모델 해시
runtime: pytorch_cuda
```

`persons` 안에는 사람마다 `bbox_xyxy`, `bbox_confidence`, `keypoints`가 기록됩니다. 테스트 이미지에서 사람이 잘 안 보이면 검출 수가 0일 수 있습니다. 이때도 **YOLO GPU 추론 자체가 실패한 것은 아닙니다.**

이제 Jetson의 **실제 GPU 추론이 성공했는지** 확인했습니다. 다음 단계에서는 PC의 기존 플랫폼과 연결합니다.

---

## PART 5. PC의 기존 FastAPI와 PostgreSQL에 Jetson 결과를 연결합니다

### 실습 20. Jetson 결과를 기존 PostgreSQL에 저장하는 코드를 작성합니다

**PC 터미널**에서 진행합니다. `subject13_ai_platform`을 다시 엽니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
```

7일차 PC 컨테이너 안에 모든 Jetson GPU 패키지를 설치하지 않습니다. PC는 **Jetson에게 분석을 요청하고 받은 JSON만 검증하여 DB에 기록**합니다.

이전에 만든 `app/models.py`의 세 테이블을 재사용합니다.

```text
camera_sources       → day08-jetson-pose 입력 출처
ai_models            → YOLO11n-Pose / PT SHA256 버전
inference_events     → 실제 Pose 추론 결과의 공통 이벤트
```

새 테이블을 추가할 필요가 없으므로 `Base.metadata.drop_all()`이나 DB 초기화 명령을 실행하지 않습니다.

**의사코드**

```text
Jetson Pose의 공통 InferenceEvent 받기
                   ↓
live_inference/action인지 확인
                   ↓
jetson: 접두사의 event_id인지 검사
                   ↓
기존 DB에 같은 event_id 있는지 조회
      ├── 있으면: 기존 기록 반환 (중복 INSERT 없음)
      └── 없으면:
             입력 출처 day08-jetson-pose 생성·조회
                   ↓
             YOLO 모델 버전 생성·조회
                   ↓
             inference_events 새 행 저장
                   ↓
               저장 결과 JSON 반환
```

**파일: `app/jetson_repo.py`**

```python
"""8일차 Jetson의 실제 Pose 이벤트를 2일차 PostgreSQL 테이블에 저장합니다."""
from datetime import timezone

from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from app.models import CameraSource, InferenceEventRow
from app.repository import get_or_create_model, row_to_event
from app.schemas import InferenceEvent


def save_jetson_event(db: Session, event: InferenceEvent):
    if event.source_mode != "live_inference" or event.task != "action":
        raise ValueError("Jetson Pose의 live_inference/action 결과만 저장합니다.")
    if not event.event_id.startswith("jetson:"):
        raise ValueError("Jetson 이벤트 ID에는 jetson: 접두사가 필요합니다.")
    lookup = (select(InferenceEventRow)
              .options(selectinload(InferenceEventRow.source),
                       selectinload(InferenceEventRow.model))
              .where(InferenceEventRow.event_id == event.event_id))
    try:
        previous = db.scalar(lookup)
        if previous is not None:
            return row_to_event(previous), False
        source = db.scalar(select(CameraSource).where(CameraSource.source_key == "day08-jetson-pose"))
        if source is None:
            source = CameraSource(source_key="day08-jetson-pose",
                                  name="8일차 Jetson Orin Nano Super 이미지 Pose",
                                  source_type="image_file")
            db.add(source)
            db.flush()
        model = get_or_create_model(db, event)
        stamp = event.recorded_at
        utc = stamp.astimezone(timezone.utc) if stamp.tzinfo and stamp.utcoffset() is not None else None
        db.add(InferenceEventRow(
            event_id=event.event_id, source_id=source.id, model_id=model.id,
            task=event.task, source_mode=event.source_mode,
            source_project=event.source_project, source_path=event.source_path,
            recorded_at_raw=stamp.isoformat(), recorded_at_utc=utc,
            input_ref=event.input_ref, predicted_label=event.predicted_label,
            expected_label=event.expected_label, confidence=event.confidence,
            decision=event.decision, reasons=list(event.reasons), details=dict(event.details),
        ))
        db.commit()
        return row_to_event(db.scalar(lookup)), True
    except Exception:
        db.rollback()
        raise
```

**코드 이해하기**

`CameraSource(source_key="day08-jetson-pose")`는 기존 교과 9 OCR이나 4일차 IP Webcam과 **구분되는 출처**입니다. `get_or_create_model()`은 2일차에 작성한 함수를 그대로 사용합니다. `event_id`가 중복되면 같은 기록을 다시 저장하지 않아, API 재전송으로 중복 레코드가 생기는 일을 줄입니다.

### 실습 21. Jetson HTTP 응답을 공통 이벤트로 변환하는 PC API를 작성합니다

**이번 코드는 무엇을 하나요?** 브라우저에서 JPG/PNG를 선택하면 PC FastAPI가 그 이미지를 Jetson의 GPU API로 전송합니다. Jetson이 반환한 사람 Pose 결과를 Pydantic으로 검증하고, 1일차부터 사용한 `InferenceEvent`로 바꾸어 PostgreSQL에 저장합니다.

**의사코드**

```text
PC 브라우저에서 JPG/PNG 선택
                 ↓
POST /api/v1/jetson/analyze
                 ↓
파일 종류·크기·실제 이미지 검사
                 ↓
Jetson LAN IP:8018로 이미지와 장치 키 전송
                 ↓
Jetson이 실제 GPU 추론 후 JSON 반환
                 ↓
PoseResponse 검증 (모델 · CUDA · 사람 수 · Keypoint)
                 ↓
pose_to_event() → 공통 InferenceEvent
                 ↓
decision = UNKNOWN, reason = POSE_ONLY_NO_ACTION_DECISION
                 ↓
app/jetson_repo.py → 기존 PostgreSQL 저장
                 ↓
웹에 이벤트 JSON 반환
```

오늘은 **Action Recognition 결과를 임의로 생성하거나 샘플 라벨을 고정하지 않습니다.** Jetson의 실제 GPU에서 추론한 사람 검출 결과만 저장합니다.

**파일: `app/jetson_api.py`**

```python
"""8일차: PC FastAPI에서 Jetson GPU Pose 서비스를 호출하고 기존 DB에 저장합니다."""
import io
import os
from datetime import datetime
from pathlib import Path
from uuid import UUID

import httpx
from fastapi import APIRouter, Depends, File, HTTPException, Query, UploadFile
from fastapi.responses import FileResponse
from PIL import Image, UnidentifiedImageError
from pydantic import BaseModel, ConfigDict, Field, AwareDatetime
from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from app.db import get_db
from app.jetson_repo import save_jetson_event
from app.models import InferenceEventRow
from app.repository import row_to_event
from app.schemas import InferenceEvent

router = APIRouter(tags=["day08-jetson"])
ROOT = Path(__file__).resolve().parents[1]
MAX_IMAGE_BYTES = 5 * 1024 * 1024


class PersonPose(BaseModel):
    bbox_xyxy: list[float] = Field(min_length=4, max_length=4)
    bbox_confidence: float = Field(ge=0, le=1)
    keypoints: list[list[float | None]] = Field(default_factory=list, max_length=17)


class PoseResponse(BaseModel):
    model_config = ConfigDict(extra="forbid")
    event_id: UUID
    recorded_at: AwareDatetime
    input_ref: str
    width: int = Field(gt=0)
    height: int = Field(gt=0)
    num_persons: int = Field(ge=0, le=5)
    persons: list[PersonPose] = Field(max_length=5)
    inference_ms: float = Field(ge=0)
    device_id: str
    model_id: str
    model_sha256: str = Field(pattern=r"^[0-9a-f]{64}$")
    runtime: str
    gpu_name: str
    note: str


def jetson_connection() -> tuple[str, str]:
    url = os.getenv("DAY08_JETSON_URL", "").rstrip("/")
    key = os.getenv("DAY08_JETSON_API_KEY", "")
    if not url.startswith("http://") or len(key) < 24:
        raise HTTPException(503, "PC의 DAY08_JETSON_URL/API_KEY 설정을 확인하세요.")
    return url, key


def pose_to_event(pose: PoseResponse) -> InferenceEvent:
    if len(pose.persons) != pose.num_persons:
        raise ValueError("Jetson 응답의 사람 수와 사람 목록 길이가 다릅니다.")
    if pose.model_id != "yolo11n-pose" or pose.runtime != "pytorch_cuda":
        raise ValueError("8일차의 YOLO11n Pose/CUDA 응답이 아닙니다.")
    if any(len(person.keypoints) not in (0, 17) for person in pose.persons):
        raise ValueError("Pose Keypoint가 17개로 구성되지 않았습니다.")
    strongest = max((person.bbox_confidence for person in pose.persons), default=None)
    # Pose Detection을 6개 Action 인식이나 산업안전 OK/NG로 해석하지 않습니다.
    reason = "POSE_ONLY_NO_ACTION_DECISION" if pose.persons else "NO_PERSON_DETECTED"
    return InferenceEvent(
        event_id=f"jetson:{pose.event_id}", task="action", source_mode="live_inference",
        source_project="subject13_ai_platform", source_path="artifacts/day05/jetson/yolo11n-pose.pt",
        recorded_at=pose.recorded_at,
        input_ref=f"{pose.device_id}/{Path(pose.input_ref).name}",
        processor_id=pose.model_id, processor_version=f"day08-cuda-{pose.model_sha256[:12]}",
        predicted_label="person" if pose.persons else None,
        expected_label=None, confidence=strongest, decision="UNKNOWN", reasons=[reason],
        details={"device_kind": "jetson_orin_nano_super", "device_id": pose.device_id,
                 "gpu_name": pose.gpu_name, "model_sha256": pose.model_sha256,
                 "inference_ms": pose.inference_ms, "runtime": pose.runtime,
                 "input_width": pose.width, "input_height": pose.height,
                 "num_persons": pose.num_persons,
                 "persons": [person.model_dump(mode="json") for person in pose.persons],
                 "note": pose.note},
    )


@router.get("/jetson")
def jetson_page():
    return FileResponse(ROOT / "web" / "jetson.html")


@router.get("/api/v1/jetson/health")
def jetson_health():
    url, _ = jetson_connection()
    try:
        with httpx.Client(timeout=3) as client:
            resp = client.get(url + "/health")
            resp.raise_for_status()
            result = resp.json()
        return {"connected": result.get("status") == "ready" and result.get("gpu_available") is True,
                "runtime": result.get("runtime"), "gpu_name": result.get("gpu_name")}
    except (httpx.HTTPError, ValueError):
        return {"connected": False, "runtime": None, "gpu_name": None}


@router.post("/api/v1/jetson/analyze")
async def jetson_analyze(image: UploadFile = File(...), db: Session = Depends(get_db)):
    url, key = jetson_connection()
    if image.content_type not in {"image/jpeg", "image/png"}:
        raise HTTPException(415, "JPG/PNG 이미지만 업로드할 수 있습니다.")
    raw = await image.read(MAX_IMAGE_BYTES + 1)
    if not raw or len(raw) > MAX_IMAGE_BYTES:
        raise HTTPException(413, "빈 파일이나 5MB 초과 파일은 허용하지 않습니다.")
    try:
        with Image.open(io.BytesIO(raw)) as original:
            width, height = original.size
            if width * height > 4096 * 4096:
                raise HTTPException(413, "이미지 해상도가 너무 큽니다.")
            original.verify()
    except (UnidentifiedImageError, OSError, ValueError):
        raise HTTPException(422, "유효한 JPG/PNG 파일인지 확인하세요.") from None
    filename = Path((image.filename or "pose.jpg").replace("\\", "/")).name[:100]
    try:
        with httpx.Client(timeout=65) as client:
            response = client.post(url + "/api/v1/jetson/pose",
                                   headers={"X-Jetson-Key": key},
                                   files={"image": (filename, raw, image.content_type)})
        response.raise_for_status()
        pose = PoseResponse.model_validate(response.json())
        event = pose_to_event(pose)
    except httpx.RequestError as error:
        raise HTTPException(503, "Jetson 추론 서버에 연결할 수 없습니다.") from error
    except (httpx.HTTPStatusError, ValueError, KeyError) as error:
        raise HTTPException(502, "Jetson 서버의 추론 응답이 정상적이지 않습니다.") from error
    stored, created = save_jetson_event(db, event)
    return {"saved": created, "event": stored.model_dump(mode="json")}


@router.get("/api/v1/jetson/events", response_model=list[InferenceEvent])
def jetson_events(limit: int = Query(default=20, ge=1, le=100), db: Session = Depends(get_db)):
    statement = (select(InferenceEventRow)
                 .options(selectinload(InferenceEventRow.source),
                          selectinload(InferenceEventRow.model))
                 .where(InferenceEventRow.event_id.like("jetson:%"))
                 .order_by(InferenceEventRow.id.desc()).limit(limit))
    return [row_to_event(row) for row in db.scalars(statement).all()]
```

**코드 이해하기**

- `jetson_connection()`은 PC에 설정된 Jetson 주소와 장치 키를 확인합니다. 값이 없을 때 테스트용 JSON으로 결과를 대체하지 않고 HTTP 503을 반환합니다.
- `httpx.Client(...).post()`는 PC에서 Jetson의 실제 GPU 서비스를 호출합니다.
- `PoseResponse`는 GPU 실행 여부, 모델 이름, 사람 수·BBox·Keypoint, SHA256을 담은 Jetson 응답이 올바른지 검사합니다. 잘못된 응답은 DB에 저장하지 않습니다.
- `pose_to_event()`는 Pose 검출과 Action 분류를 혼동하지 않도록 `UNKNOWN`을 사용합니다. 사람을 검출했다고 해서 정상작업 또는 위험행동이라고 판정하지 않습니다.
- `save_jetson_event()`는 2일차 DB를 재사용하며, 중복 `event_id`의 INSERT를 막습니다.
- 이미지 바이트는 **PC→Jetson 요청에만 사용**하고 PostgreSQL에 저장하지 않습니다. 저장하는 것은 분석 메타데이터입니다.

`/api/v1/jetson/health`는 Jetson API에 연결되는지 확인합니다. `/api/v1/jetson/events`는 이번 실습에서 저장한 `jetson:` 이벤트만 조회합니다. 6일차 Raspberry Pi 이벤트와 혼동하지 않습니다.

### 실습 22. 기존 `app/main.py`에 Jetson Router를 추가합니다

**수정 파일: `app/main.py`**

1~7일차에서 작성한 `app = FastAPI(...)`와 `db_router`, `pipeline_router`, `live_router`, `edge_router`, `edge_device_router`, `container_router`를 **모두 유지**합니다. 파일 **맨 아래**에 다음 두 줄만 추가합니다.

```python
from app.jetson_api import router as jetson_router
app.include_router(jetson_router)
```

이미 동일한 Router가 있다면 중복으로 추가하지 않습니다. 기존 `InferenceEvent`, DB 테이블·API 경로를 변경하지 않습니다.

### 실습 23. PC의 `.env`와 7일차 `compose.yaml`에 Jetson 주소·키를 연결합니다

**PC 터미널**에서 기존 `.env`를 엽니다.

```bash
cd ~/ai_vision/subject13_ai_platform
nano .env
```

기존 `DB_HOST`, `DB_PASSWORD`, `REDIS_URL`, `DAY06_DEVICE_API_KEY`를 변경하거나 삭제하지 않습니다. 파일 맨 아래에 다음 두 항목을 **추가**합니다.

```dotenv
# 8일차: Jetson Orin Nano Super GPU Pose (교실 실제 LAN 주소로 수정)
DAY08_JETSON_URL=http://192.168.0.31:8018
DAY08_JETSON_API_KEY=Jetson_env_jetson과_정확히_같은_24자이상_키
```

주소와 키를 실제 값으로 변경합니다. URL은 **PC에서 Jetson으로 접근 가능한 실제 주소**여야 하며, Jetson에 설정한 `JETSON_API_KEY`와 PC의 `DAY08_JETSON_API_KEY` 값은 일치해야 합니다. 비밀번호를 소스 파일이나 Git에 기록하지 마세요.

이제 **수정 파일: 기존 루트 `compose.yaml`**에서 `services: → api: → environment:` 아래에 **다음 두 항목을 추가**합니다.

```yaml
      DAY08_JETSON_URL: ${DAY08_JETSON_URL:-}
      DAY08_JETSON_API_KEY: ${DAY08_JETSON_API_KEY:-}
```

기존 `DB_HOST: db`, `REDIS_URL: redis://redis:6379/0` 등 **이전에 작성한 환경변수를 그대로 둡니다**. `worker`의 환경설정은 Jetson HTTP 요청이 필요하지 않으므로 변경하지 않습니다.

`:-`는 변수 값이 준비되지 않았을 때 빈 문자열을 사용한다는 뜻입니다. Jetson이 꺼진 날에도 **7일차의 DB·Redis·Worker가 중단되지 않게 하기 위한 설정**입니다. 대신 Jetson 연결 API는 설정이 없다면 준비되지 않았다고 명확히 응답합니다.

설정 문법을 확인합니다. `docker compose config` 전체 출력에는 환경변수·비밀번호가 포함될 수 있으므로 `--quiet`를 사용합니다.

```bash
docker compose config --quiet
```

### 실습 24. PC 컨테이너에 새 기능이 포함될 위치를 확인합니다

이번에 추가한 `app/jetson_api.py`, `app/jetson_repo.py`, 앞으로 작성할 `web/jetson.html`은 **7일차 Dockerfile의 기존 `COPY app/`, `COPY web/`에 자동으로 포함**됩니다. 따라서 7일차 루트 `Dockerfile`을 새로 만들거나, 기존 서버를 지금 중단할 필요는 없습니다.

**다음 실습 25에서 웹 파일까지 작성한 뒤 실습 26에서 단 한 번만 PC 이미지를 빌드·재실행합니다.** 현재는 기존 `db`, `redis`가 유지되고, 앞의 `docker compose config --quiet`를 통과한 상태인지 확인하면 됩니다.

```bash
docker compose ps
```

기존 DB 볼륨을 초기화하거나 `docker compose down -v`를 실행하지 않습니다.

---

## PART 6. 웹에서 실제 이미지·Skeleton을 확인하고 PostgreSQL 기록을 검증합니다

### 실습 25. Jetson Pose 결과를 표시할 웹 화면을 작성합니다

**이번 웹 화면은 무엇을 하나요?** 학생이 이미지 파일을 선택하면 PC FastAPI를 경유하여 Jetson GPU 모델로 분석하고, 원본 이미지 위에 Bounding Box와 17개 Keypoint를 Canvas로 표시합니다. 결과의 신뢰도·추론 시간·저장된 공통 JSON·최근 이력도 확인합니다.

**의사코드**

```text
JPG/PNG를 파일 선택창에서 선택
                ↓
브라우저에서 선택한 이미지를 미리보기
                ↓
PC FastAPI로 이미지 전송
                ↓
Jetson Pose 결과 + PostgreSQL 저장 결과 받기
                ↓
Bounding Box·Skeleton을 원본 이미지 위에 그리기
                ↓
Person Confidence·추론 시간·공통 JSON 확인
                ↓
최근 Jetson 이벤트 목록 갱신
```

**파일: `web/jetson.html`**

```html
<!doctype html>
<html lang="ko">
<head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>교과 13 · 8일차 Jetson GPU Pose</title>
<style>
body{font-family:system-ui,sans-serif;max-width:1100px;margin:26px auto;padding:0 16px;color:#26344a}
.panel{border:1px solid #ccd4df;border-radius:10px;padding:15px;margin:12px 0}
button,input{font:inherit;padding:8px}button{cursor:pointer}
canvas{max-width:100%;height:auto;display:block;background:#eee;border:1px solid #ddd}
pre{background:#f2f4f8;padding:12px;white-space:pre-wrap;overflow-wrap:anywhere}
small{color:#65758a}.err{color:#9b2435}table{border-collapse:collapse;width:100%}td,th{padding:7px;border-bottom:1px solid #ddd;text-align:left}
</style>
</head>
<body>
<h1>교과 13 · 8일차 Jetson Orin Nano Super</h1>
<p>교과 10의 <strong>YOLO11n-Pose를 Jetson GPU Docker</strong>에서 실행합니다. 오늘의 결과는 사람과 Keypoint이며, 6개 Action 분류 또는 산업안전 위험 판정이 아닙니다.</p>
<p><a href="/">1일차 기록</a> · <a href="/db">2일차 DB</a> · <a href="/containers">7일차 Docker</a> · <a href="/docs">Swagger</a></p>
<div class="panel"><button id="health">Jetson 연결 확인</button> <span id="connection">연결 확인 전</span></div>
<div class="panel">
<label for="photo">사람이 나온 JPG/PNG 이미지 선택</label>
<input id="photo" type="file" accept="image/jpeg,image/png">
<button id="run">Jetson GPU에서 분석하고 DB에 저장</button>
<p id="status" role="status">파일을 선택하세요.</p>
<canvas id="canvas" width="640" height="360"></canvas>
<small>초록색: Bounding Box · 노란색: COCO 17 Keypoint·Skeleton · 파일은 DB에 원본 이미지로 저장하지 않습니다.</small>
</div>
<div class="panel"><h2>저장한 결과</h2><pre id="result">아직 분석 결과가 없습니다.</pre></div>
<div class="panel"><h2>최근 Jetson 추론 이력</h2><button id="historyBtn">기록 다시 불러오기</button><table><thead><tr><th>시간</th><th>이미지</th><th>사람 수</th><th>추론 시간</th><th>판정</th></tr></thead><tbody id="rows"></tbody></table></div>
<script>
const $=id=>document.getElementById(id);
const edges=[[5,7],[7,9],[6,8],[8,10],[5,6],[5,11],[6,12],[11,12],[11,13],[13,15],[12,14],[14,16],[0,1],[0,2],[1,3],[2,4]];
let selectedImage=null;
$('photo').addEventListener('change',()=>{
  const file=$('photo').files[0]; selectedImage=null;
  if(!file)return;
  const url=URL.createObjectURL(file); const im=new Image();
  im.onload=()=>{URL.revokeObjectURL(url); selectedImage=im;
    const c=$('canvas');c.width=im.naturalWidth;c.height=im.naturalHeight;
    c.getContext('2d').drawImage(im,0,0);};
  im.onerror=()=>{URL.revokeObjectURL(url);$('status').textContent='이미지 파일을 열지 못했습니다.'};
  im.src=url;
});
function draw(persons){
  if(!selectedImage)return;
  const c=$('canvas'),ctx=c.getContext('2d');ctx.drawImage(selectedImage,0,0);
  ctx.lineWidth=Math.max(2,c.width/350);
  for(const person of persons){
    ctx.strokeStyle='#18b65a'; const [x1,y1,x2,y2]=person.bbox_xyxy;
    ctx.strokeRect(x1,y1,x2-x1,y2-y1);
    ctx.fillStyle='#18b65a';ctx.font=`${Math.max(14,c.width/45)}px sans-serif`;
    ctx.fillText('person '+person.bbox_confidence.toFixed(2),x1,Math.max(16,y1-7));
    const points=person.keypoints||[];ctx.strokeStyle='#ffcc18';ctx.fillStyle='#ffcc18';
    for(const [a,b] of edges){
      const p=points[a],q=points[b];
      if(!p||!q||p[2]===null||q[2]===null||p[2]<0.3||q[2]<0.3)continue;
      ctx.beginPath();ctx.moveTo(p[0],p[1]);ctx.lineTo(q[0],q[1]);ctx.stroke();
    }
    for(const p of points){if(!p||p[2]===null||p[2]<0.3)continue;
      ctx.beginPath();ctx.arc(p[0],p[1],Math.max(2.5,c.width/200),0,Math.PI*2);ctx.fill();}
  }
}
async function check(){
  try{const r=await fetch('/api/v1/jetson/health');const x=await r.json();
    $('connection').textContent=x.connected?`연결됨 · ${x.gpu_name} · ${x.runtime}`:'Jetson 연결 확인 필요';}
  catch(err){$('connection').textContent='PC API가 응답하지 않습니다.'}
}
async function history(){try{
  const r=await fetch('/api/v1/jetson/events');if(!r.ok)throw Error(`HTTP ${r.status}`);
  const entries=await r.json();$('rows').replaceChildren();
  for(const e of entries){const tr=document.createElement('tr');
    for(const value of [e.recorded_at,e.input_ref,e.details.num_persons,e.details.inference_ms+'ms',e.decision]){
      const td=document.createElement('td');td.textContent=String(value);tr.append(td);} $('rows').append(tr);}
}catch(err){$('status').textContent='기록 조회 오류: '+err.message}}
$('run').addEventListener('click',async()=>{
  const file=$('photo').files[0];if(!file)return $('status').textContent='이미지를 먼저 선택하세요.';
  $('status').textContent='Jetson GPU 분석 요청 중...';
  try{const form=new FormData();form.append('image',file);
    const response=await fetch('/api/v1/jetson/analyze',{method:'POST',body:form});
    const data=await response.json();if(!response.ok)throw Error(JSON.stringify(data));
    draw(data.event.details.persons); $('result').textContent=JSON.stringify(data,null,2);
    $('status').textContent=`저장 ${data.saved?'완료':'중복 요청'} · 사람 ${data.event.details.num_persons}명 · ${data.event.details.inference_ms}ms · Pose만 분석`;await history();
  }catch(err){$('status').textContent='실패: '+err.message;}
});
$('health').addEventListener('click',check);$('historyBtn').addEventListener('click',history);
check();history();
</script>
</body>
</html>
```

**웹 코드 이해하기**

`fetch('/api/v1/jetson/analyze', ...)`는 이미지 원본을 PC 서버에 전송합니다. PC가 Jetson과 통신하므로 브라우저에는 Jetson의 비밀 키가 필요하지 않습니다. `Canvas`는 원본 이미지의 실제 픽셀 크기와 `bbox_xyxy`, Keypoint 좌표를 사용하여 덧그립니다. COCO 17개의 연결은 `edges` 목록으로 정의합니다.

`decision: UNKNOWN`이 나와도 이번 실습에서 검출한 Person BBox·Keypoint가 정상이라면 **정상적인 Pose 서비스 결과**입니다. 오늘은 위험행동의 OK/NG를 평가하는 수업이 아니기 때문입니다.

### 실습 26. PC 웹브라우저에서 실제 Jetson 추론을 실행합니다

이제 PC의 `app/`와 `web/` 코드가 모두 완성되었습니다. **이번 단계에서 한 번만 7일차 PC 이미지를 재빌드**하여 변경사항을 적용합니다. `db`, `redis` 및 기존 PostgreSQL named volume은 유지합니다.

```bash
docker compose build api
docker compose up -d --force-recreate api worker
docker compose ps
```

PC 브라우저에서 `http://127.0.0.1:8000/docs`를 열고 `GET /api/v1/jetson/health`, `POST /api/v1/jetson/analyze`, `GET /api/v1/jetson/events` 세 경로가 등록되었는지 확인합니다.

```bash
curl --fail http://127.0.0.1:8000/api/v1/jetson/health
```

`connected: true`이면 PC에서 Jetson에 연결할 수 있다는 뜻입니다. `connected: false`이면 PC와 Jetson의 LAN 주소·Docker 포트·컨테이너 로그·방화벽을 확인합니다.

**PC 브라우저**에서 다음 주소를 엽니다.

```text
http://127.0.0.1:8000/jetson
```

1. **Jetson 연결 확인** 버튼을 눌러 `연결됨`인지 확인합니다.
2. 교과 10에서 사용한 사람이 나온 JPG/PNG를 선택합니다.
3. **Jetson GPU에서 분석하고 DB에 저장**을 누릅니다.
4. 녹색 Bounding Box·노란 Keypoint/Skeleton과 `num_persons`, `inference_ms`를 확인합니다.
5. 표시된 JSON의 `event_id`가 `jetson:`으로 시작하고 `source_mode`가 `live_inference`인지 확인합니다.
6. **최근 Jetson 추론 이력**에 방금 실행한 이미지 파일명이 나타나는지 확인합니다.

처음 분석한 인물 이미지에서 Pose를 검출하지 못하면 입력 이미지의 사람 크기·가림·방향 등을 확인합니다. **검출된 사람 수가 0인 응답도 GPU 추론이 실제 실행된 결과**입니다. 다만 Skeleton 시각화 확인을 위해서는 Keypoint가 정상 검출되는 이미지를 사용해야 합니다.

웹 대신 터미널에서도 PC API를 직접 호출해 봅니다. **PC 터미널**에서 다음을 실행합니다.

```bash
curl --fail --max-time 80 -X POST \
  http://127.0.0.1:8000/api/v1/jetson/analyze \
  -F "image=@deploy/day08_jetson/samples/test_pose.jpg;type=image/jpeg" \
  | python3 -m json.tool
```

PC 터미널에 `deploy/day08_jetson/samples/test_pose.jpg`가 있어야 위 명령이 실행됩니다. 결과 JSON의 주요 항목은 `saved`, `event`, `event.details.persons`, `event.details.inference_ms`입니다. 다른 이미지를 보내면 새로운 이벤트 ID가 생성되고 기존 결과를 보존합니다.

### 실습 27. 기존 PostgreSQL에 Jetson 이벤트가 저장되었는지 확인합니다

먼저 FastAPI에서 Jetson 이력만 조회합니다.

```bash
curl --fail http://127.0.0.1:8000/api/v1/jetson/events \
  | python3 -m json.tool
```

DB에서 직접 확인하려면 PC의 기존 PostgreSQL 컨테이너에 접속하여 읽기 쿼리만 실행합니다.

```bash
docker compose exec db sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT event_id, task, source_mode, predicted_label, decision FROM inference_events WHERE event_id LIKE '\''jetson:%'\'' ORDER BY id DESC LIMIT 10;"'
```

**확인할 내용**

| DB 항목 | 오늘 기대하는 의미 |
|---|---|
| `event_id` | `jetson:` + Jetson이 생성한 실제 UUID |
| `task` | `action` (Pose 데이터를 나중에 Action Recognition으로 연결할 예정) |
| `source_mode` | `live_inference` (오늘 실제 Jetson GPU가 새로 추론한 결과) |
| `predicted_label` | 사람이 검출되면 `person`, 아니면 SQL `NULL` |
| `decision` | `UNKNOWN` (6개 동작을 아직 판정하지 않음) |
| `details` | Keypoint 좌표, BBox, GPU, 모델 SHA256, 추론 시간 |

이 기록은 교과 9 OCR의 OK/NG와 달리 **사람 검출 결과**입니다. 화면이나 DB의 UNKNOWN을 보고 AI 모델이 잘못되었다고 판단하지 않습니다.

PC Docker 상태도 다시 확인합니다.

```bash
curl --fail http://127.0.0.1:8000/api/v1/containers/status
docker compose ps
```

`postgresql`, `redis`, `celery_worker_responding`이 이전처럼 정상인지 확인합니다. 이번 Jetson 연결 때문에 7일차의 네 서비스를 삭제하거나 새로 만들지 않았습니다.

---

## PART 7. 자동 테스트·문제 해결·작업 정리

### 실습 28. Jetson API 연결과 DB 저장 코드를 자동 테스트합니다

실제 Jetson 하드웨어가 켜지지 않아도 **데이터 계약·중복 저장·입력 검증·오류 응답**은 자동으로 검사할 수 있습니다. 하지만 자동 테스트만으로 실제 CUDA 추론 성공을 증명할 수는 없습니다. 실제 GPU는 앞의 실습 18~19·26에서 확인합니다.

**이번 테스트에서 확인할 흐름**

```text
Jetson 응답 필드가 올바른가?
           ↓
Pose를 위험행동/Action 판정으로 오해하지 않는가?
           ↓
사람 검출이 없을 때도 UNKNOWN이 유지되는가?
           ↓
잘못된 모델·CUDA 응답을 거부하는가?
           ↓
DB에 중복 event_id가 들어가지 않는가?
           ↓
이미지가 아닌 파일·과도한 파일을 거부하는가?
           ↓
PC Jetson 분석 API와 기록 조회 API가 연결되는가?
```

**파일: `tests/test_day08.py`**

```python
"""8일차: 실제 장비는 필요 없는 API/DB/응답 규격 검증."""
import io
import hashlib
import uuid
from datetime import datetime, timezone

import pytest
from PIL import Image
from fastapi import FastAPI
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import Session
from sqlalchemy.pool import StaticPool

from app.db import Base, get_db
from app.jetson_api import router, PoseResponse, pose_to_event
from app.jetson_repo import save_jetson_event


def sample_result(event_id=None, persons=True):
    people = ([{"bbox_xyxy": [1, 2, 90, 110], "bbox_confidence": 0.94,
                "keypoints": [[float(i), float(i + 1), .91] for i in range(17)]}]
              if persons else [])
    return {
        "event_id": event_id or str(uuid.uuid4()), "recorded_at": "2026-10-17T09:00:00+00:00",
        "input_ref": "arms_up_person_a.jpg", "width": 120, "height": 150,
        "num_persons": len(people), "persons": people, "inference_ms": 42.4,
        "device_id": "jetson-orin-nano-super", "model_id": "yolo11n-pose",
        "model_sha256": "a" * 64, "runtime": "pytorch_cuda", "gpu_name": "Jetson Orin",
        "note": "Pose only",
    }


@pytest.fixture()
def client_and_db(monkeypatch):
    engine = create_engine("sqlite://", connect_args={"check_same_thread": False},
                           poolclass=StaticPool)
    Base.metadata.create_all(engine)
    app = FastAPI()
    app.include_router(router)
    def db_override():
        with Session(engine) as db:
            yield db
    app.dependency_overrides[get_db] = db_override
    client = TestClient(app)
    return client, engine


def png_file():
    f = io.BytesIO()
    Image.new("RGB", (128, 96), (200, 150, 50)).save(f, "PNG")
    return f.getvalue()


def setup_fake_jetson(monkeypatch, result, status_code=200):
    import app.jetson_api as module
    monkeypatch.setattr(module, "jetson_connection", lambda: ("http://jetson-lan:8018", "x" * 32))
    class RemoteReply:
        def __init__(self): self.status_code = status_code
        def raise_for_status(self):
            if status_code >= 400:
                import httpx
                raise httpx.HTTPStatusError("Error", request=httpx.Request("POST","http://jetson-lan"),
                                            response=httpx.Response(status_code))
        def json(self): return result
    class FakeClient:
        def __init__(self, **kw): pass
        def __enter__(self): return self
        def __exit__(self, *args): pass
        def post(self, url, **kwargs):
            assert url.endswith("/api/v1/jetson/pose")
            assert kwargs["headers"]["X-Jetson-Key"] == "x" * 32
            assert kwargs["files"]["image"][1]
            return RemoteReply()
        def get(self, *args, **kwargs): return RemoteReply()
    monkeypatch.setattr(module.httpx, "Client", FakeClient)


def test_pose_is_unknown_not_action():
    e = pose_to_event(PoseResponse.model_validate(sample_result()))
    assert e.task == "action" and e.decision == "UNKNOWN"
    assert e.predicted_label == "person" and e.confidence == .94
    assert e.reasons == ["POSE_ONLY_NO_ACTION_DECISION"]
    assert len(e.details["persons"][0]["keypoints"]) == 17


def test_no_person_is_unknown_no_confidence():
    e = pose_to_event(PoseResponse.model_validate(sample_result(persons=False)))
    assert e.predicted_label is None and e.confidence is None
    assert e.reasons == ["NO_PERSON_DETECTED"]


def test_inconsistent_person_count_rejected():
    response = sample_result()
    response["num_persons"] = 0
    with pytest.raises(ValueError, match="사람 수"):
        pose_to_event(PoseResponse.model_validate(response))


def test_incorrect_model_or_runtime_rejected():
    response = sample_result()
    response["runtime"] = "cpu"
    with pytest.raises(ValueError, match="CUDA"):
        pose_to_event(PoseResponse.model_validate(response))


def test_db_first_save_then_duplicate(client_and_db):
    _, engine = client_and_db
    item = pose_to_event(PoseResponse.model_validate(sample_result()))
    with Session(engine) as db:
        first, inserted = save_jetson_event(db, item)
        second, inserted_again = save_jetson_event(db, item)
    assert inserted is True and inserted_again is False
    assert second.event_id == first.event_id


def test_actual_api_image_upload_and_history(client_and_db, monkeypatch):
    client, _ = client_and_db
    setup_fake_jetson(monkeypatch, sample_result())
    response = client.post("/api/v1/jetson/analyze",
                           files={"image": ("pose.png", png_file(), "image/png")})
    assert response.status_code == 200, response.text
    item = response.json()
    assert item["saved"] is True
    assert item["event"]["source_mode"] == "live_inference"
    assert item["event"]["decision"] == "UNKNOWN"
    assert len(client.get("/api/v1/jetson/events").json()) == 1
    assert client.get("/jetson").status_code == 200


def test_remote_bad_response_returns_502_and_no_db_save(client_and_db, monkeypatch):
    client, _ = client_and_db
    bad = sample_result()
    bad["model_sha256"] = "not-valid"
    setup_fake_jetson(monkeypatch, bad)
    resp = client.post("/api/v1/jetson/analyze",
                       files={"image": ("p.png", png_file(), "image/png")})
    assert resp.status_code == 502
    assert client.get("/api/v1/jetson/events").json() == []


def test_unsupported_upload_rejected_before_remote_call(client_and_db, monkeypatch):
    client, _ = client_and_db
    setup_fake_jetson(monkeypatch, sample_result())
    r = client.post("/api/v1/jetson/analyze",
                    files={"image": ("sample.txt", b"hello", "text/plain")})
    assert r.status_code == 415


def test_fake_image_with_png_mime_rejected(client_and_db, monkeypatch):
    client, _ = client_and_db
    setup_fake_jetson(monkeypatch, sample_result())
    r = client.post("/api/v1/jetson/analyze",
                    files={"image": ("broken.png", b"not-an-image", "image/png")})
    assert r.status_code == 422


def test_upload_limit(client_and_db, monkeypatch):
    client, _ = client_and_db
    setup_fake_jetson(monkeypatch, sample_result())
    r = client.post("/api/v1/jetson/analyze",
                    files={"image": ("big.png", b"A" * (5*1024*1024+1), "image/png")})
    assert r.status_code == 413


def test_url_and_key_must_be_configured(monkeypatch):
    from app.jetson_api import jetson_connection
    monkeypatch.delenv("DAY08_JETSON_URL", raising=False)
    monkeypatch.delenv("DAY08_JETSON_API_KEY", raising=False)
    from fastapi import HTTPException
    with pytest.raises(HTTPException) as exc:
        jetson_connection()
    assert exc.value.status_code == 503


def test_reject_wrong_task_for_jetson_repo(client_and_db):
    _, engine = client_and_db
    item = pose_to_event(PoseResponse.model_validate(sample_result()))
    wrong = item.model_copy(update={"task": "inspection"})
    with Session(engine) as db:
        with pytest.raises(ValueError):
            save_jetson_event(db, wrong)
```

테스트 파일을 저장한 뒤 PC의 **기존 가상환경**에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m pytest -q tests/test_day08.py
```

**이 교재의 8일차 코드에서 확인한 결과**는 `12 passed`입니다. 이 테스트에서는 Jetson의 네트워크 응답·GPU 하드웨어를 테스트용 대역으로 교체하고 DB는 임시 SQLite를 사용했습니다. **실제 Jetson GPU 성능이나 실제 PostgreSQL 서버 상태를 측정한 결과는 아닙니다.**

7일차와 8일차 코드가 함께 동작하는지도 확인합니다.

```bash
python -m pytest -q tests/test_day07.py tests/test_day08.py
```

이 두 신규 일차의 코드에서 **20개 테스트 통과**를 확인했습니다. 마지막으로 학생 PC의 1~6일차 실제 코드를 포함한 전체 테스트도 실행합니다.

```bash
python -m pytest -q
```

전체 테스트는 학생이 실제로 작성한 이전 일차의 파일·모델·의존성에 영향을 받을 수 있으므로 오류가 있다면 어떤 테스트가 실패했는지부터 확인합니다. **전체 테스트를 한 번 실행했다고 실제 카메라·Jetson·Raspberry Pi·Docker 하드웨어 검증을 대신한 것은 아닙니다.**

### 실습 29. 오류가 발생하면 문제가 시작된 지점부터 확인합니다

| 증상 | 확인할 곳 | 해결 방법 |
|---|---|---|
| Jetson에서 `uname -m`이 `aarch64`가 아님 | Jetson 터미널인지, PC 터미널인지 | 실제 Jetson 장치로 접속해서 실행 |
| `docker compose` 없음 | Jetson Docker 설치 | JetPack에 맞는 Docker Compose v2 준비 |
| `runtime=nvidia` 오류 | NVIDIA Container Runtime | Jetson 패키지·`nvidia-ctk`·Docker 설정 확인 |
| `torch.cuda.is_available() == False` | JetPack·이미지·Runtime | GPU 사용이 가능한 컨테이너인지 먼저 확인, CPU로 숨기지 않음 |
| `ModuleNotFoundError: ultralytics` | Jetson의 이미지 태그 | JetPack 6용 Ultralytics 이미지를 사용했는지 확인 |
| 이미지 빌드 또는 `pip` 실패 | 네트워크·패키지 저장소 | 승인된 온라인/오프라인 설치 환경, JetPack 버전 확인 |
| `.pt` 파일 또는 SHA256 불일치 | `artifacts/day05/jetson` | 5일차 실제 모델 재확인 후 재전송 |
| Jetson 컨테이너 재시작 반복 | `logs pose` | 모델 누락·CUDA 불가·환경변수 불일치 확인 |
| Jetson `/health` 연결 실패 | Jetson LAN IP·포트·컨테이너 | 실제 바인딩 IP, Uvicorn, 방화벽·네트워크 격리 확인 |
| PC `/api/v1/jetson/health`의 `connected:false` | PC→Jetson HTTP | PC Docker 컨테이너에서 Jetson LAN IP 접근 가능한지 확인 |
| HTTP 401 | Jetson 장치 인증 키 | `.env.jetson`과 PC `.env`의 키 일치 확인 |
| HTTP 413 또는 415 | 업로드 크기·파일 종류 | 5MB 이하 JPG/PNG 실제 이미지로 다시 확인 |
| HTTP 422 | 파일 손상·JSON 검증 실패 | 파일을 정상 이미지로 저장하고 실제 응답 필드 확인 |
| HTTP 502/503 | Jetson 추론/연결 실패 | Jetson 로그→PC API 로그→네트워크 순서 확인 |
| Bounding Box는 나오지만 Skeleton 없음 | Keypoint 품질/Confidence | 실제 17개 좌표가 생성되었는지 JSON과 촬영 조건 확인 |
| 웹 페이지가 404 | `web/jetson.html`·Router | `app/main.py`의 `include_router`와 PC 이미지 재빌드 확인 |
| DB 이력 없음 | PostgreSQL·Repo·API 결과 | `/api/v1/jetson/events`와 PC API 로그 및 DB 상태 확인 |

**문제 해결 순서**

```text
5일차 PT/ONNX/해시
       ↓
Jetson ARM64·JetPack·NVIDIA Runtime
       ↓
Jetson Docker GPU CUDA=True
       ↓
Jetson /health 및 실제 JPG 추론
       ↓
PC → Jetson 네트워크·인증 키
       ↓
PC FastAPI /api/v1/jetson/analyze
       ↓
PostgreSQL 이벤트 저장
       ↓
웹 Canvas 표시
```

Jetson 서버가 꺼져 있거나 네트워크가 끊기면 **거짓 샘플 응답을 정상 결과인 것처럼 저장하지 않습니다.** 문제를 해결한 후 실제 이미지를 다시 전송합니다.

### 실습 30. 8일차 실행 방법을 운영 문서에 남깁니다

**파일: `docs/day08_jetson_runbook.md`**

아래 문서 전체를 작성합니다.

~~~~markdown
# 교과 13 · 8일차 Jetson Orin Nano Super 운영 메모

## 구성

- PC: 7일차 `compose.yaml`의 `api`·`db`·`redis`·`worker` 유지
- Jetson: `deploy/day08_jetson/compose.jetson.yaml`의 GPU `pose` 서비스
- 모델: 5일차 `artifacts/day05/jetson/yolo11n-pose.pt`
- 결과: PC `inference_events` 테이블에 `jetson:` 이벤트 저장
- 판정: Pose 단계이므로 `UNKNOWN`으로 기록. 산업안전 위험 판정이 아님

## 실행 순서

1. PC에서 `docker compose ps`로 4서비스 확인
2. Jetson에서 모델 SHA256과 `torch.cuda.is_available()` 확인
3. Jetson에서 `docker compose --env-file .env.jetson -f deploy/day08_jetson/compose.jetson.yaml up -d pose`
4. Jetson `/health`, 이미지 1장 추론 확인
5. PC `.env`의 `DAY08_JETSON_URL`·`DAY08_JETSON_API_KEY` 확인
6. PC에서 `docker compose build api && docker compose up -d --force-recreate api worker`
7. PC 웹 `/jetson`에서 이미지 추론·DB 저장 확인

## 장애 시 확인

- Jetson GPU: `torch.cuda.is_available()` / `tegrastats`
- Jetson 로그: `docker compose --env-file .env.jetson -f deploy/day08_jetson/compose.jetson.yaml logs --tail=70 pose`
- PC 로그: `docker compose logs --tail=70 api`
- PC 이력: `GET /api/v1/jetson/events`
- PC 7일차 상태: `GET /api/v1/containers/status`

## 금지 사항

- `.env`·`.env.jetson`, 비밀 키, 실제 모델, 사람 사진을 외부 Git에 올리지 않음
- 기존 7일차 DB 볼륨 삭제하지 않음 (`docker compose down -v` 금지)
- 오늘의 Pose Detection을 6개 Action Classification 또는 OK/NG로 오해하지 않음

## 9일차 연결

동일한 실제 YOLO11n-Pose PT/ONNX를 Jetson에서 TensorRT로 최적화하고,
이미지 입력을 실시간 카메라 프레임 스트림으로 확장해 지연·FPS·복구를 비교합니다.
~~~~

### 실습 31. 기존 README와 구조도를 8일차로 업데이트합니다

**수정 파일: `README.md`**

기존 1~7일차 실행 내용은 그대로 두고 아래 내용을 문서 끝에 추가합니다.

~~~~markdown
## 8일차 — Jetson GPU Pose 추론·PC 결과 저장

- 모델: 5일차의 `yolo11n-pose.pt` 재사용
- Jetson: JetPack 6 / ARM64 / NVIDIA Runtime / PyTorch CUDA
- Jetson 컨테이너: `deploy/day08_jetson/compose.jetson.yaml`
- PC 연결: `app/jetson_api.py`, `app/jetson_repo.py`
- PC 웹: `http://127.0.0.1:8000/jetson`
- PC API: `/api/v1/jetson/health`, `/api/v1/jetson/analyze`, `/api/v1/jetson/events`
- DB: 기존 `inference_events`를 재사용하며 `jetson:` 접두사로 출처 구분
- 주의: 지금은 이미지 한 장의 Pose만 분석, Action Recognition과 TensorRT는 미구현
- 상세 절차: `docs/day08_jetson_runbook.md`
~~~~

**수정 파일: `docs/architecture.md`**

기존 1~7일차 구조 설명 맨 아래에 다음 내용을 추가합니다.

~~~~markdown
## 8일차 — Jetson Orin Nano Super GPU 서비스

브라우저 → PC FastAPI `/api/v1/jetson/analyze`
           → Jetson GPU Docker `POST /api/v1/jetson/pose`
           → YOLO11n-Pose (`yolo11n-pose.pt`)
           → Person BBox·COCO 17 Keypoint JSON
           → PC 공통 `InferenceEvent`
           → 기존 PostgreSQL `inference_events`
           → 웹 Canvas·최근 이력

- 7일차 PC `api/db/redis/worker` 구성은 그대로 유지
- Jetson 컨테이너는 PC의 PostgreSQL/Redis를 별도로 복제하지 않음
- 8일차에는 Action 분류나 OK/NG 판단, TensorRT 엔진 생성은 수행하지 않음
- 9일차에 TensorRT·연속 카메라·지연/FPS로 확장
~~~~

### 실습 32. Git에 올리지 않을 파일을 다시 확인합니다

이미 `.gitignore`에 `.env`, `*.pt`, `*.onnx`, `artifacts/` 등을 제외했다면 기존 규칙을 유지합니다. 이번에 추가한 Jetson 실제 설정과 학생 촬영 이미지도 제외되도록 **기존 `.gitignore` 맨 아래에 다음 내용을 추가**합니다.

```gitignore
# 8일차 Jetson 로컬 설정·모델·인물 이미지
.env.jetson
deploy/day08_jetson/samples/
artifacts/day05/jetson/
# 이전 .env.* 제외 규칙의 영향을 받는 공개용 설정 예시는 허용
!deploy/day08_jetson/.env.jetson.example
```

`.env.jetson.example`은 실제 키가 없는 안내 파일이므로 Git에 기록할 수 있습니다. `.env.jetson`과 학생 실제 이미지는 Git에 올리지 않습니다. 루트 `.env`도 1~7일차처럼 제외합니다.

```bash
git status --short
git check-ignore .env .env.jetson artifacts/day05/jetson/yolo11n-pose.pt \
  deploy/day08_jetson/samples/test_pose.jpg || true
```

모델 파일·비밀 키·개인 이미지가 실수로 Staged에 올라왔다면 **커밋 전에 반드시 제외**합니다. `.gitignore`는 이전에 이미 추적된 파일을 자동으로 제거하지 않습니다.

---

# 6. 오늘 작성한 내용의 최종 정리 및 GitHub 업로드

## 6.1. 오늘 무엇을 완성했나요?

오늘은 기존 PC의 네 컨테이너와 PostgreSQL을 유지하면서 **Jetson Orin Nano Super에서 실제 YOLO11n-Pose 모델을 GPU Docker로 실행**했습니다. PC와 Jetson이 하나의 JSON 계약으로 연결되도록 FastAPI를 확장했으며, 실제 이미지에서 얻은 Bounding Box·COCO 17 Keypoint·추론 시간을 웹 화면에 표시하고 **기존 `inference_events`에 저장**하도록 구성했습니다.

```text
교과 10의 실제 YOLO11n-Pose PT
                 ↓
Jetson JetPack 6 / ARM64 GPU Docker
                 ↓
JPG/PNG → 실제 Person Pose 추론
                 ↓
Bounding Box · COCO 17 · Inference Time
                 ↓
PC FastAPI + 장치 인증·Pydantic 검증
                 ↓
공통 InferenceEvent (source_mode=live_inference)
                 ↓
2일차부터 사용한 PostgreSQL
                 ↓
웹 `/jetson`의 Skeleton·최근 이력 확인
                 ↓
9일차: ONNX/TensorRT·카메라 연속 입력 확장
```

**지금까지 만든 플랫폼은 그대로 하나입니다.** PC의 FastAPI·PostgreSQL·Redis·Celery를 새로 만들지 않고, Jetson의 GPU 분석 능력을 외부 서비스로 추가한 것입니다.

## 6.2. 최종 자가 체크리스트

- [ ] 7일차 PC Docker Compose의 `api`, `db`, `redis`, `worker`가 유지되어 있습니다.
- [ ] 5일차 실제 `yolo11n-pose.pt`, `.onnx`, `pose_export_manifest.json`이 있습니다.
- [ ] `19_check_day08_inputs.py`에서 모델 SHA256을 검사하고 `day08_manifest.json`을 만들었습니다.
- [ ] Jetson에서 ARM64·JetPack 6·NVIDIA Container Runtime을 확인했습니다.
- [ ] Jetson 컨테이너에서 `torch.cuda.is_available() == True`를 확인했습니다.
- [ ] Jetson의 `/api/v1/jetson/pose`가 실제 이미지·Bounding Box·Keypoint를 반환했습니다.
- [ ] PC의 `/api/v1/jetson/health`가 Jetson 연결 상태를 보여 줍니다.
- [ ] PC 웹 `/jetson`에서 실제 이미지·BBox·Skeleton을 확인했습니다.
- [ ] PostgreSQL에 `jetson:` 이벤트가 저장되었고 `decision=UNKNOWN`의 의미를 설명할 수 있습니다.
- [ ] 8일차 자동 테스트가 통과했습니다.
- [ ] README·구조도·운영 문서를 업데이트했습니다.
- [ ] 비밀 키·실제 모델·사람 이미지가 Git 업로드 대상에 들어 있지 않습니다.

## 6.3. 오늘 추가한 소스코드를 Git에 기록합니다

**PC 터미널**에서 기존 교과 13 프로젝트의 Git을 그대로 사용합니다. 새 저장소를 `git init`으로 다시 만들지 않습니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git status
```

오늘 작성한 파일만 Stage 합니다.

```bash
git add app/jetson_api.py app/jetson_repo.py app/main.py \
  scripts/19_check_day08_inputs.py \
  deploy/day08_jetson/app_jetson.py \
  deploy/day08_jetson/requirements.jetson.txt \
  deploy/day08_jetson/Dockerfile.jetson \
  deploy/day08_jetson/compose.jetson.yaml \
  deploy/day08_jetson/.env.jetson.example \
  web/jetson.html tests/test_day08.py \
  docs/day08_jetson_runbook.md docs/architecture.md \
  compose.yaml README.md .gitignore

git status
```

`.env`, `.env.jetson`, PT/ONNX, `day08_manifest.json`, 학생 이미지가 Staged 목록에 **없어야 합니다**. 확인한 뒤 커밋합니다.

```bash
git commit -m "교과13 8일차: Jetson ARM64 GPU Pose 배포 및 PC PostgreSQL 연동"
git log -1 --oneline
git status
```

기존 원격 저장소가 설정되어 있고 기관에서 외부 업로드를 허용한다면 다음을 실행합니다.

```bash
git remote -v
git push origin main
```

폐쇄망이거나 외부 GitHub 사용이 금지된 경우 **로컬 Git 또는 승인된 내부 GitLab/Gitea**에만 기록합니다. 실제 모델·이미지·기밀 키는 원격 저장소에 업로드하지 않습니다.

## 6.4. 9일차 수업과 어떻게 연결되나요?

8일차에는 교과 10의 YOLO11n-Pose 모델이 **Jetson GPU Docker에서 실제 이미지 한 장을 분석하고, PC 플랫폼에서 결과를 기록하는 기능**까지 완성했습니다.

9일차에는 오늘 배포한 동일한 모델을 바꾸어 학습하지 않고, **5일차에 준비한 `yolo11n-pose.onnx`를 Jetson 환경에서 TensorRT로 최적화**합니다. 그런 다음 이미지 1장 입력에서 **IP Webcam/USB 카메라 연속 Frame 입력**으로 확장하여 추론 지연, End-to-End FPS, 네트워크·카메라 장애 대응을 비교하게 됩니다.

**다음 수업에서도 오늘 만든 PC FastAPI, 기존 PostgreSQL, Jetson 이미지·API 코드와 모델 Manifest를 그대로 사용합니다.**

---

## 공식 기술 참고

- [Ultralytics — Jetson 장치별 YOLO Docker 및 GPU 실행 가이드](https://docs.ultralytics.com/guides/nvidia-jetson)
- [Ultralytics — Docker Quickstart: JetPack 전용 이미지 구분](https://docs.ultralytics.com/guides/docker-quickstart)
- [NVIDIA — Jetson Orin Nano Developer Kit Docker Setup](https://docs.nvidia.com/jetson/orin-nano-devkit/user-guide/latest/setup_docker.html)

공식 안내에서 JetPack 6 GPU 이미지를 구분하는 원칙을 참고했습니다. 실제 이미지와 컨테이너 도구가 정상 동작하는지는 **수업에 사용할 Jetson의 JetPack·네트워크·모델 버전으로 직접 확인**해야 합니다.

