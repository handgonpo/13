## 1. 오늘의 목표

5일차에는 직접 촬영한 `normal`, `warning_red` 이미지로 TinyCNN을 학습하고, PyTorch 모델을 ONNX로 변환했습니다. 또한 `runtime.py`, `training_config.json`, `manifest.json`과 함께 Raspberry Pi용 실행 파일을 준비했습니다.

오늘은 이 **동일한 ONNX 모델과 전처리 설정**을 Raspberry Pi에 전달하여 현장 장치에서 직접 실행합니다. Raspberry Pi에서 수행한 결과를 PC의 기존 FastAPI로 보내고, 2일차에 만든 PostgreSQL 테이블과 연결합니다.

수업을 마치면 다음을 직접 수행할 수 있습니다.

- [ ] 5일차 배포 폴더의 `tinycnn.onnx`와 Manifest·SHA256을 확인합니다.
- [ ] Raspberry Pi OS, ARM64, Python·ONNX Runtime·카메라·GPIO 환경을 확인합니다.
- [ ] 5일차에 사용한 전처리·클래스·Threshold를 Raspberry Pi에서도 그대로 사용합니다.
- [ ] 정상 카드는 **파란 LED**, 빨간 표시 카드는 **빨간 LED**, `UNKNOWN` 또는 오류는 **LED 모두 OFF**로 처리합니다.
- [ ] Raspberry Pi가 만든 결과 JSON을 기존 교과 13 FastAPI에 전달합니다.
- [ ] PC PostgreSQL의 기존 `camera_sources`, `ai_models`, `inference_events`에 장치 결과를 저장하고 웹에서 조회합니다.
- [ ] 네트워크가 끊기면 결과를 Pi에 저장하고 복구 후 재전송합니다.
- [ ] 카메라 단절·낮은 신뢰도·프로세스 종료를 구분하여 안전한 LED 상태를 유지합니다.
- [ ] 자동 테스트, 운영 메모, README, Git 커밋으로 수업을 마무리합니다.

**오늘 완성할 결과:** `USB 카메라 → Raspberry Pi ONNX Runtime CPU → normal / warning_red / UNKNOWN → GPIO → 로컬 전송 대기함 → PC FastAPI → PostgreSQL → 웹 화면`.

5일차에 준비한 배포 패키지와 카메라·GPIO는 실제 장비가 있어야 마지막까지 검증할 수 있습니다. 모델이 준비되지 않았다면 아래 준비물 확인에서 멈춘 뒤 5일차 결과물을 확보합니다.

---

## 2. 오늘 수업의 전체 흐름

![[Pasted image 20261018130723.png]]

```text
[교과 13 PC : 1~5일차 프로젝트 유지]
    │
    ├─ 2일차 PostgreSQL / 3일차 Redis·Celery / 4일차 OCR·Pose
    ├─ 5일차 tinycnn.onnx + training_config.json + manifest.json
    │                         ↓
    │                SHA256 / 클래스 / 크기 검사
    │                         ↓
    └─ 6일차 장치 결과 수신 API 작성 ─────────────────────────┐
                                                           │
[5일차 배포 파일을 Raspberry Pi로 안전하게 전송]              │
                        ↓                                  │
             Raspberry Pi ARM64 확인                       │
                        ↓                                  │
          Python .venv + ONNX Runtime CPU                  │
                        ↓                                  │
              USB 카메라 Frame 읽기                         │
                        ↓                                  │
           RGB / Resize 64×64 / float32                    │
                        ↓                                  │
                 TinyCNN ONNX 추론                          │
                        ↓                                  │
              OK / NG / UNKNOWN 결정                       │
                        ↓                                  │
          GPIO 파랑 / 빨강 / 모두 OFF                        │
                        ↓                                  │
              이벤트 UUID + UTC 시간                        │
                        ↓                                  │
            Pi outbox/*.json 에 먼저 저장                    │
                        ↓                                  │
              HTTP POST + 장치 인증 ────────────────────────┘
                        ↓
          PC FastAPI /api/v1/edge-device/events
                        ↓
       2일차 PostgreSQL inference_events 저장
                        ↓
             PC /edge-device 웹 기록 조회
                        ↓
     카메라·네트워크 오류 재현 → 재전송·복구·테스트
                        ↓
               README · Git 기록 → 7일차 Docker
```

**구분해서 이해합니다.** 오늘의 `inference_ms`는 **Pi에서 ONNX 모델을 실행하는 시간**입니다. 카메라 캡처·JPEG 인코딩·HTTP 전송·PostgreSQL 저장까지 포함한 전체 지연시간과 같지 않습니다. 또한 3일차의 Redis·Celery는 계속 유지하지만, **이번 Pi 결과 전송에서는 로컬 outbox와 HTTP POST를 사용**합니다. 추론 시간을 줄이기 위해 Redis·Celery를 쓰는 것으로 설명하지 않습니다.

---

## 3. 실습 전에 알아야 할 핵심

### 3.1. 이전 교과와 1~5일차에서 가져올 결과

교과 9~12를 완료한 상태에서 교과 13을 진행하므로, 오늘도 어제 사용한 **동일한** `~/ai_vision/subject13_ai_platform`을 이어 사용합니다. 교과 9의 OCR과 교과 10의 Action 모델을 다시 만드는 날은 아닙니다. 5일차에 추가한 **교육용 2클래스 TinyCNN**만 Raspberry Pi로 옮깁니다.

| 이전 실습 | 필요한 파일/기능 | 6일차에서 쓰는 이유 |
|---|---|---|
| 교과 13 · 2일차 | `app/db.py`, `app/models.py`, `app/repository.py`, PostgreSQL | 장치 결과를 기존 테이블에 저장 |
| 교과 13 · 4일차 | `app/schemas.py`의 `live_inference` 허용 | 새 장치 이벤트를 공통 JSON으로 표현 |
| 교과 13 · 5일차 | `deploy/day05_raspberrypi/tinycnn.onnx` | Raspberry Pi에서 실행할 ONNX 모델 |
| 교과 13 · 5일차 | `training_config.json`, `manifest.json` | 학습 시 클래스·전처리·Threshold 및 모델 무결성 확인 |
| 교과 13 · 5일차 | `runtime.py`, `infer_on_device.py` | PyTorch 없이 CPU 추론, 이미지 1장 확인 |
| 교과 13 · 5일차 | `data/day05/images/test/` 중 정상·빨간 카드 각 1장 | Pi에서 카메라 연결 전에 기능 검사 |

5일차 배포 결과가 없다면 다음 **다섯 파일을 동일한 폴더 구조로** 제공받아 실습할 수 있습니다. 단, 서로 다른 학생의 ONNX와 `training_config.json`을 섞어 사용하면 안 됩니다. **하나의 학습 실행에서 생성된 5개 파일을 한 묶음으로** 사용하세요.

```text
deploy/day05_raspberrypi/
├── tinycnn.onnx
├── training_config.json
├── manifest.json
├── runtime.py
└── infer_on_device.py
```

오늘 실습에는 교과 9 OCR 이미지 전체, 교과 10 Random Forest 모델, 교과 10 MP4가 직접 필요하지 않습니다. 이전 프로젝트의 원본을 지우지 않습니다.

### 3.2. Raspberry Pi에서 실행하는 코드와 PC에서 실행하는 코드를 구분합니다

| 장치 | 실행할 프로그램 | 담당 작업 |
|---|---|---|
| PC(개발용 Ubuntu/WSL) | 기존 교과 13 FastAPI·PostgreSQL | 장치 결과 인증·수신·DB 저장·웹 조회 |
| Raspberry Pi | `run_camera.py` + `runtime.py` | 카메라·ONNX CPU·LED·전송 대기함 |
| 브라우저 | `/edge-device` | Pi가 보낸 결과 확인 |

**Pi에는 PostgreSQL·Redis·Celery·교과 9 OCR·교과 10 YOLO Pose를 설치하지 않습니다.** 해당 서비스는 계속 PC의 기존 교과 13 환경에서 실행됩니다.

### 3.3. 5일차 모델의 클래스와 전처리를 유지합니다

```text
카메라 BGR Frame
     ↓
OpenCV에서 JPEG 변환
     ↓
5일차 runtime.py : PIL 이미지 읽기 → RGB
     ↓
64×64 Bilinear Resize
     ↓
float32 / 255.0
     ↓
NCHW : [1, 3, 64, 64]
     ↓
ONNX 입력 image → 출력 logits
     ↓
Softmax → normal / warning_red
     ↓
최대 확률이 confidence_threshold_demo 미만이면 UNKNOWN
```

`warning_red`는 **빨간 카드가 보이는 교육용 상태**일 뿐 현장 불량·위험 확정을 뜻하지 않습니다. 카메라가 끊기거나 추론 함수가 실패했을 때에는 어떤 카드를 보았다고 추정하지 않고 `UNKNOWN`으로 처리합니다.

### 3.4. GPIO는 BCM 번호로 연결합니다

| 신호 | BCM 번호 | 보드 물리 핀 번호 | 결과 |
|---|---:|---:|---|
| 파란 LED | GPIO17 | 11 | `OK`일 때 ON |
| 빨간 LED | GPIO27 | 13 | `NG`일 때 ON |
| GND | — | 6 | 두 LED 음극의 공통 접지 |

연결할 때 각 LED마다 **전류 제한 저항 330Ω**을 직렬로 사용합니다. 예를 들어 `GPIO17 → 330Ω 저항 → 파란 LED의 긴 다리(애노드) → 짧은 다리(캐소드) → GND`로 연결합니다. **전원을 끈 상태에서 배선**하고, GPIO에 5V를 연결하지 않습니다. 모터·릴레이 등 큰 부하는 GPIO에 직접 연결하지 않습니다.

| 상황 | 파란 LED | 빨간 LED |
|---|---|---|
| `normal` / `OK` | ON | OFF |
| `warning_red` / `NG` | OFF | ON |
| 낮은 Confidence / `UNKNOWN` | OFF | OFF |
| 카메라 연결 실패 | OFF | OFF |
| 추론 오류 / 정상 종료 | OFF | OFF |

### 3.5. PC와 Raspberry Pi의 통신은 한 방향으로 시작합니다

```text
Pi 추론 결과 → Pi outbox/*.json → POST /api/v1/edge-device/events
                                     ↓
                         PC 인증 키 확인 → 기존 DB
```

장치 이벤트에는 `event_id`, `device_id`, `captured_at`, 클래스, Confidence, 판정, 모델 ID·버전·SHA256, 추론시간만 전송합니다. **원본 사람 영상이나 카드 이미지 자체를 전송하지 않습니다.**

- 서버가 응답하지 않으면 `outbox/`의 이벤트 JSON을 지우지 않습니다.
- 서버가 저장에 성공하면 해당 JSON만 삭제합니다.
- 동일한 `event_id`가 두 번 전송되어도 **DB에는 한 번만 저장**합니다.
- 장치 인증용 키는 PC `.env`와 Pi `.env`에 각각 보관하고 Git에 올리지 않습니다.
- HTTP와 공유 키 예제는 **접근이 제한된 교육용 내부망**을 전제로 합니다. 공개 인터넷에서는 HTTPS, 네트워크 접근 제어, 키 교체 등 추가 보안이 필요합니다.

### 3.6. Python 패키지는 PC와 Raspberry Pi가 다릅니다

PC에는 5일차의 PyTorch·ONNX 개발환경이 남아 있습니다. Raspberry Pi는 학습하지 않고 ONNX Runtime **CPU**로 추론합니다. Raspberry Pi OS 64비트의 실제 Python 버전과 호환되는 **ARM64 ONNX Runtime Wheel**이 있어야 하므로, 일반 x86_64 패키지를 그대로 복사하지 않습니다.

`--dry-run`은 GPIO만 가상 LED로 대체하며, **ONNX 추론은 실제로 실행**합니다. 개발 PC에서는 이미지 한 장으로 전송 코드까지 점검할 수 있고, 실제 USB 카메라와 GPIO는 Raspberry Pi에서 확인합니다.

---

## 4. 오늘 사용할 기술 스택

| 기술 | 이번 실습에서 맡는 역할 |
|---|---|
| Raspberry Pi OS 64-bit · SSH | 장치 환경과 원격 접속 |
| Python `.venv` · `pathlib` · `json` | Raspberry Pi 프로그램 실행·설정 관리 |
| ONNX Runtime CPU · Pillow · NumPy | 5일차 TinyCNN 추론과 동일한 RGB 전처리 |
| OpenCV | USB 카메라 프레임 캡처·JPEG 인코딩 |
| GPIO Zero · lgpio | BCM17·BCM27의 파란색/빨간색 LED 제어 |
| SHA256 · Manifest | 전송한 모델·설정 일치 확인 |
| 로컬 Outbox · HTTP POST | 통신 장애 시 유실 방지와 자동 재전송 |
| FastAPI · Pydantic | PC에서 장치 결과 JSON 검증·API 제공 |
| SQLAlchemy · PostgreSQL | 2일차 테이블에 Pi 추론 기록 저장 |
| HTML · JavaScript `fetch()` | PC 화면에서 새 장치 기록 조회 |
| Pytest · Git | 코드 검증, 변경 이력 관리 |

---

# 5. 처음부터 끝까지 따라 하는 상세 실습

## PART 1. 5일차 프로젝트와 Raspberry Pi 배포물을 확인합니다

### 실습 1. 전날 프로젝트를 다시 엽니다

PC의 WSL Ubuntu 터미널에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
pwd
which python
git status --short
```

`pwd`에 `subject13_ai_platform`, `which python`에 `.venv/bin/python`이 표시되는지 확인합니다. **새 프로젝트를 만들지 않습니다.** 5일차 소스코드와 FastAPI Router를 그대로 유지합니다.

다음 파일들이 있는지 확인합니다.

```bash
ls -lh app/main.py app/db.py app/models.py app/repository.py app/schemas.py
ls -lh app/edge_api.py edge/runtime.py
find deploy/day05_raspberrypi -maxdepth 1 -type f | sort
```

`tinycnn.onnx`, `training_config.json`, `manifest.json`, `runtime.py`, `infer_on_device.py` 다섯 파일이 없으면 **5일차 배포 패키지 생성 작업**을 완료하거나 제공된 한 묶음의 배포 파일을 배치합니다.

```bash
# 5일차 ONNX 변환·검증을 완료한 경우에만 다시 패키징합니다.
python -m scripts.14_package_day05
```

5일차의 `tinycnn.onnx`는 아직 이 교재에서 다시 학습하거나 바꾸지 않습니다. **5일차 ONNX 검증이 통과한 모델**을 사용해야 합니다.

### 실습 2. 6일차 폴더를 추가합니다

다음 명령은 기존 파일을 삭제하지 않고 폴더만 생성합니다.

```bash
mkdir -p device/day06_pi scripts tests web docs deploy/day06_raspberrypi
# 장치 코드를 Python 모듈로 관리하기 위한 빈 파일
touch device/__init__.py device/day06_pi/__init__.py
```

오늘 추가할 코드의 위치를 확인합니다.

```text
subject13_ai_platform/
├── app/
│   ├── main.py                 [기존 유지, Router 2줄 추가]
│   ├── db.py · models.py · repository.py · schemas.py [기존 그대로]
│   ├── edge_device_repo.py     [6일차 신규: DB 저장]
│   └── edge_device_api.py      [6일차 신규: 장치 결과 수신/조회]
├── device/
│   ├── __init__.py             [빈 파일]
│   └── day06_pi/
│       ├── __init__.py         [빈 파일]
│       ├── gpio_led.py         [신규]
│       ├── outbox.py           [신규]
│       ├── verify_package.py   [신규]
│       ├── run_camera.py       [신규]
│       ├── device_config.example.json [신규]
│       └── requirements_pi.txt [신규]
├── scripts/16_package_day06.py [신규: 5일차 모델+6일차 실행파일 복사]
├── web/pi.html                 [신규: Pi 결과 조회 화면]
├── tests/test_day06.py         [신규: API·큐·GPIO 검증]
├── docs/day06_pi_runbook.md    [신규: 실행·복구 절차]
├── deploy/
│   ├── day05_raspberrypi/     [5일차 배포본 원본 유지]
│   └── day06_raspberrypi/     [6일차에 생성, Git 제외]
└── .env                        [2~5일차 설정 유지, 6일차 키 추가]
```

### 실습 3. Pi 접속과 장비 준비 상태를 확인합니다

Raspberry Pi를 네트워크에 연결하고 SSH를 사용할 수 있도록 준비합니다. Pi OS를 새로 설치해야 한다면 **64-bit Raspberry Pi OS와 학생 계정·SSH 접근 설정**을 먼저 완료합니다.

PC 터미널에서 Pi의 실제 주소로 접속합니다.

```bash
ssh <Pi의_사용자계정>@<Pi_IP주소>
```

예시 형식은 `ssh student@192.168.0.30`입니다. 실제 계정·주소는 장치에서 확인한 값으로 바꿉니다. Pi 터미널에서 실행합니다.

```bash
uname -m
cat /etc/os-release | head -6
python3 --version
hostname -I
ls -l /dev/video* 2>/dev/null || true
```

- `uname -m`이 **`aarch64`**이면 64비트 ARM 환경입니다.
- `/dev/video0` 등이 보이면 USB 카메라가 인식된 후보입니다. 실제 영상 프레임을 읽는지는 뒤에서 다시 확인합니다.
- Pi 카메라 모듈(CSI)을 사용한다면 장치 접근 방식이 USB UVC 카메라와 다릅니다. **이번 공통 실습은 USB UVC 카메라 기준**으로 작성했습니다.
- SSH 계정명 `pi`가 기본으로 존재한다고 가정하지 않습니다.

터미널이 PC인지 Pi인지 헷갈리지 않도록 **PC에서 실행할 명령과 Pi에서 실행할 명령을 구분**해 계속 진행합니다.

---

## PART 2. PC에 장치 결과 수신 API와 PostgreSQL 저장 기능을 추가합니다

### 실습 4. Raspberry Pi 이벤트를 기존 DB에 저장하는 함수를 작성합니다

**이번 코드의 목적**은 새 테이블을 만들지 않고, 2일차의 `CameraSource`, `AIModel`, `InferenceEventRow`에 Pi 결과를 저장하는 것입니다. 4일차의 `save_live_event()`를 삭제하거나 수정하지 않습니다. Raspberry Pi 전용 입력 출처 이름을 분리하기 위해 새 함수를 추가합니다.

**의사코드**

```text
Pi 결과 → 공통 InferenceEvent
       ↓
event_id가 이미 있는지 확인
       ├─ 있으면 기존 결과 반환 (중복 저장 X)
       └─ 없으면 계속
              ↓
       day06-pi-{device_id} 입력 출처 찾기/생성
              ↓
       기존 AIModel 찾기/생성
              ↓
       inference_events 1건 INSERT + COMMIT
              ↓
       저장된 공통 JSON 응답
```

**파일: `app/edge_device_repo.py`**

```python
"""6일차: Raspberry Pi 이벤트를 기존 2일차 PostgreSQL 구조로 저장."""
from datetime import timezone
from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload
from app.models import AIModel, CameraSource, InferenceEventRow
from app.repository import get_or_create_model, row_to_event
from app.schemas import InferenceEvent


def save_pi_event(db: Session, event: InferenceEvent, device_id: str):
    if event.source_mode != "live_inference" or event.task != "inspection":
        raise ValueError("Pi의 live_inference 검사 이벤트만 저장합니다.")
    lookup = (select(InferenceEventRow)
              .options(selectinload(InferenceEventRow.source),
                       selectinload(InferenceEventRow.model))
              .where(InferenceEventRow.event_id == event.event_id))
    try:
        existing = db.scalar(lookup)
        if existing is not None:
            return row_to_event(existing), False
        source_key = f"day06-pi-{device_id}"
        source = db.scalar(select(CameraSource).where(CameraSource.source_key == source_key))
        if source is None:
            source = CameraSource(source_key=source_key,
                                  name=f"6일차 Raspberry Pi {device_id}",
                                  source_type="usb_camera")
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
            decision=event.decision, reasons=list(event.reasons), details=dict(event.details)))
        db.commit()
        return row_to_event(db.scalar(lookup)), True
    except Exception:
        db.rollback()
        raise
```

`save_pi_event()`는 2일차의 `get_or_create_model()`과 `row_to_event()`를 그대로 사용합니다. 따라서 기존 ORM 테이블 구조와 충돌하지 않습니다. 같은 UUID가 재전송되어도 한 건만 저장합니다. 저장 실패 시 `rollback()`하여 부분 기록을 남기지 않습니다.

### 실습 5. Pi 장치 인증·JSON 검증·조회 API를 작성합니다

**이번 코드는 무엇을 하나나요?** Raspberry Pi가 보내는 JSON을 Pydantic으로 검사하고, 공유 키가 맞는 장치 요청만 DB에 저장합니다. 잘못된 `decision`과 `predicted_label` 조합을 거절하고, 이미 저장된 Pi 기록을 조회할 API도 만듭니다.

**의사코드**

```text
POST /api/v1/edge-device/events
       ↓
X-Device-Key가 PC 설정과 같은가?
       ├─ 아니면 401
       └─ 같으면 계속
              ↓
UUID / UTC 시각 / 클래스 / Confidence / SHA256 검사
              ↓
normal이면 OK, warning_red이면 NG, 검출 불가면 UNKNOWN 확인
              ↓
공통 InferenceEvent(live_inference) 생성
              ↓
PostgreSQL 저장·중복 검사
              ↓
{"saved": true/false, "event": {...}}
```

**파일: `app/edge_device_api.py`**

```python
"""6일차: Raspberry Pi CPU ONNX 추론 결과를 기존 플랫폼 DB에 등록·조회."""
import os
import secrets
from datetime import datetime
from pathlib import Path
from typing import Literal
from uuid import UUID

from fastapi import APIRouter, Depends, Header, HTTPException, Query
from fastapi.responses import FileResponse
from pydantic import AwareDatetime, BaseModel, Field, ConfigDict, model_validator
from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from app.db import get_db
from app.edge_device_repo import save_pi_event
from app.models import CameraSource, InferenceEventRow
from app.repository import row_to_event
from app.schemas import InferenceEvent

router = APIRouter(tags=["day06-raspberrypi"])
ROOT = Path(__file__).resolve().parents[1]


class PiInferencePayload(BaseModel):
    model_config = ConfigDict(extra="forbid")
    event_id: UUID
    device_id: str = Field(min_length=3, max_length=40, pattern=r"^[A-Za-z0-9_-]+$")
    captured_at: AwareDatetime
    input_ref: str = Field(min_length=1, max_length=100, pattern=r"^[A-Za-z0-9_.:-]+$")
    predicted_label: Literal["normal", "warning_red"] | None = None
    confidence: float | None = Field(default=None, ge=0, le=1)
    decision: Literal["OK", "NG", "UNKNOWN"]
    reason_code: Literal["NORMAL", "WARNING_RED", "LOW_CONFIDENCE", "CAMERA_UNAVAILABLE", "INFERENCE_ERROR"]
    inference_ms: float | None = Field(default=None, ge=0, le=60000)
    model_id: str = Field(min_length=1, max_length=80)
    model_version: str = Field(min_length=1, max_length=80)
    model_sha256: str = Field(pattern=r"^[0-9a-f]{64}$")

    @model_validator(mode="after")
    def consistent_result(self):
        rules = {"NORMAL": ("OK", "normal"),
                 "WARNING_RED": ("NG", "warning_red")}
        if self.reason_code in rules:
            expected_decision, expected_label = rules[self.reason_code]
            if (self.decision != expected_decision or self.predicted_label != expected_label
                    or self.confidence is None or self.inference_ms is None):
                raise ValueError("판정, 예측 라벨, 신뢰도 또는 추론시간이 일치하지 않습니다.")
        elif self.reason_code == "LOW_CONFIDENCE":
            if self.decision != "UNKNOWN" or self.predicted_label is None or self.confidence is None:
                raise ValueError("LOW_CONFIDENCE는 예측 라벨/신뢰도와 UNKNOWN이 필요합니다.")
        else:
            if self.decision != "UNKNOWN" or self.predicted_label is not None or self.confidence is not None:
                raise ValueError("카메라/추론 오류는 예측·신뢰도 없이 UNKNOWN이어야 합니다.")
        return self


def require_device_key(x_device_key: str | None = Header(default=None, alias="X-Device-Key")) -> None:
    configured = os.getenv("DAY06_DEVICE_API_KEY", "")
    if len(configured) < 24:
        raise HTTPException(status_code=503, detail="PC의 DAY06_DEVICE_API_KEY 설정이 필요합니다.")
    if x_device_key is None or not secrets.compare_digest(x_device_key, configured):
        raise HTTPException(status_code=401, detail="장치 인증 키를 확인하세요.")


def to_event(item: PiInferencePayload) -> InferenceEvent:
    return InferenceEvent(
        event_id=f"pi:{item.event_id}", task="inspection",
        source_mode="live_inference", source_project="subject13_ai_platform",
        source_path="deploy/day06_raspberrypi/tinycnn.onnx",
        recorded_at=item.captured_at, input_ref=f"{item.device_id}/{item.input_ref}",
        processor_id=item.model_id, processor_version=item.model_version,
        predicted_label=item.predicted_label, expected_label="normal",
        confidence=item.confidence, decision=item.decision,
        reasons=[item.reason_code],
        details={"device_kind": "raspberry_pi", "device_id": item.device_id,
                 "model_sha256": item.model_sha256,
                 "inference_ms": item.inference_ms,
                 "source_input": item.input_ref, "purpose": "day06_two_card_education"},
    )


@router.get("/api/v1/edge-device/health")
def device_health():
    return {"service": "ready", "auth_configured": len(os.getenv("DAY06_DEVICE_API_KEY", "")) >= 24,
            "purpose": "educational_two_card_classifier"}


@router.post("/api/v1/edge-device/events")
def ingest_edge_event(payload: PiInferencePayload,
                      _: None = Depends(require_device_key),
                      db: Session = Depends(get_db)):
    event, created = save_pi_event(db, to_event(payload), payload.device_id)
    return {"saved": created, "event": event.model_dump(mode="json")}


def recent_pi_events(db: Session, limit: int = 30):
    statement = (select(InferenceEventRow)
                 .join(CameraSource, CameraSource.id == InferenceEventRow.source_id)
                 .options(selectinload(InferenceEventRow.source), selectinload(InferenceEventRow.model))
                 .where(CameraSource.source_key.like("day06-pi-%"))
                 .order_by(InferenceEventRow.id.desc()).limit(limit))
    return [row_to_event(row) for row in db.scalars(statement).all()]


@router.get("/api/v1/edge-device/events", response_model=list[InferenceEvent])
def list_pi_events(limit: int = Query(default=30, ge=1, le=100), db: Session = Depends(get_db)):
    return recent_pi_events(db, limit)


@router.get("/edge-device")
def edge_device_page():
    return FileResponse(ROOT / "web" / "pi.html")
```

앞의 코드에서 중요한 부분을 확인합니다.

- `PiInferencePayload`의 `captured_at`은 **시간대가 명시된 값**만 허용합니다. Pi에서는 UTC 값을 전송합니다.
- `reason_code`는 `NORMAL`, `WARNING_RED`, `LOW_CONFIDENCE`, `CAMERA_UNAVAILABLE`, `INFERENCE_ERROR`를 구분합니다. 카메라 오류를 **NG 제품**이라고 오해하지 않습니다.
- `model_sha256`과 모델 버전은 추론 결과의 출처를 식별하는 정보입니다. 전송받은 해시를 장치가 기록한 값으로 보관하며, 이것만으로 장치의 신뢰성을 증명하지는 않습니다.
- `GET /api/v1/edge-device/events`에서는 6일차 Pi 출처만 조회합니다. 기존 교과 9·10의 이벤트 기록은 지우지 않습니다.
- 개인 이미지나 카메라 프레임은 서버에 업로드하지 않습니다.

### 실습 6. 1~5일차 FastAPI에 6일차 Router만 추가합니다

**수정 파일: `app/main.py` — 기존 `app = FastAPI(...)`와 모든 Router를 그대로 둔 뒤 다음 두 줄만 맨 아래에 한 번 추가합니다.**

```python
from app.edge_device_api import router as edge_device_router
app.include_router(edge_device_router)
```

기존 2일차 `db_router`, 3일차 `pipeline_router`, 4일차 `live_router`, 5일차 `edge_router`는 **삭제하지 않습니다**. `/docs`에서 기존 API도 함께 보여야 합니다.

### 실습 7. PC `.env`에 장치 전송 인증 키를 추가합니다

2일차에 만든 `.env`에는 `DB_USER`, `DB_PASSWORD` 등의 값이 있습니다. **기존 값은 지우지 않고** 맨 아래에 장치 키만 추가합니다.

PC 터미널에서 임의 키를 생성합니다.

```bash
python - <<'PY'
import secrets
print(secrets.token_urlsafe(32))
PY
```

출력된 문자열을 PC의 기존 `.env` 파일 맨 아래에 복사합니다. 다른 설정을 삭제하지 마세요.

```dotenv
DAY06_DEVICE_API_KEY=위에서_생성한_충분히_긴_비밀키
```

**이 값은 실제 예시가 아니라 자리표시자입니다.** 비밀 키를 README, 소스 코드, Git, 화면 캡처에 기록하지 않습니다. Raspberry Pi에서도 **같은 키**를 사용합니다.

### 실습 8. Pi 이벤트를 조회하는 웹 화면을 작성합니다

5일차의 `web/edge.html`은 **PC에서 ONNX를 확인하는 화면**으로 유지합니다. 이번에는 **Pi가 실행하여 PostgreSQL에 저장한 결과만 보여주는 별도 페이지**를 만듭니다.

**의사코드**

```text
브라우저 /edge-device
     ↓
GET /api/v1/edge-device/events
     ↓
DB에서 day06-pi-* 장치 결과만 조회
     ↓
시각 / Device ID / 예측 / 판정 / Confidence / ONNX 추론시간
     ↓
JSON 보기 → 공통 InferenceEvent 상세 내용 확인
```

**파일: `web/pi.html`**

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8"><meta name="viewport" content="width=device-width, initial-scale=1">
  <title>교과 13 · 6일차 Raspberry Pi Edge</title>
  <style>
    body{font-family:system-ui,sans-serif;max-width:1080px;margin:30px auto;padding:0 16px;color:#233345}
    section{border:1px solid #cbd5e1;border-radius:12px;padding:18px;margin:14px 0}
    table{width:100%;border-collapse:collapse}th,td{border-bottom:1px solid #ddd;padding:9px;text-align:left}
    th{background:#f6f8fa}pre{white-space:pre-wrap;overflow-wrap:anywhere;background:#f5f7fa;padding:13px}
    button{padding:7px 12px;cursor:pointer}.note{color:#53657a}.ok{color:#16713d}.ng{color:#b52b2b}.unknown{color:#a16c15}
  </style>
</head>
<body>
  <h1>6일차 · Raspberry Pi ONNX 결과 조회</h1>
  <p class="note">Raspberry Pi에서 TinyCNN을 실행하고 전송한 메타데이터입니다. 이미지·영상은 저장하지 않습니다. normal/ warning_red는 교육용 카드 분류이며 실제 현장 안전 판정이 아닙니다.</p>
  <section><button id="refresh">새로고침</button><span id="count"></span><p id="error" role="alert"></p></section>
  <section><h2>최근 장치 추론 이벤트</h2>
    <table><thead><tr><th>기록 시각</th><th>장치</th><th>예측</th><th>판정</th><th>Confidence</th><th>ONNX ms</th><th>상세</th></tr></thead>
    <tbody id="rows"></tbody></table>
  </section>
  <section><h2>선택한 공통 JSON</h2><pre id="detail">상세 보기를 선택하세요.</pre></section>
  <script>
    const tbody=document.getElementById('rows');
    const count=document.getElementById('count');
    const error=document.getElementById('error');
    const detail=document.getElementById('detail');
    async function refresh(){
      tbody.replaceChildren();error.textContent='';
      try{
        const response=await fetch('/api/v1/edge-device/events?limit=30',{cache:'no-store'});
        if(!response.ok)throw new Error('HTTP '+response.status);
        const events=await response.json();count.textContent=' 조회한 이벤트 '+events.length+'건';
        for(const event of events){
          const tr=document.createElement('tr');
          const values=[event.recorded_at,event.details.device_id,event.predicted_label??'검출 불가',
            event.decision,event.confidence==null?'미기록':event.confidence.toFixed(3),
            event.details.inference_ms??'-'];
          for(const value of values){const cell=document.createElement('td');cell.textContent=value;tr.append(cell);}
          tr.children[3].className=event.decision.toLowerCase();
          const cell=document.createElement('td');const button=document.createElement('button');
          button.textContent='JSON 보기';button.onclick=()=>detail.textContent=JSON.stringify(event,null,2);
          cell.append(button);tr.append(cell);tbody.append(tr);
        }
      }catch(exc){error.textContent='조회 실패: '+exc.message;}
    }
    document.getElementById('refresh').addEventListener('click',refresh);refresh();
  </script>
</body></html>
```

웹 코드에서 `fetch()`는 기존 FastAPI의 신규 이벤트 목록 API를 호출합니다. `textContent`로 화면을 채우므로, 기록에 들어 있는 문자열이 HTML로 해석되지 않습니다.

### 실습 9. PC 서버를 다시 실행하고 신규 주소를 확인합니다

**PC 터미널**에서 기존 PostgreSQL을 확인합니다.

```bash
sudo systemctl status postgresql --no-pager
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

기존의 `app.main:app` 하나만 실행합니다. 새 FastAPI 앱을 만들지 않습니다. `--host 0.0.0.0`은 Pi 같은 다른 장치에서 접근할 수 있도록 서버가 PC 네트워크의 인터페이스를 수신한다는 뜻입니다. **승인된 교육장 내부망에서만 사용**합니다.

웹브라우저에서 열어 봅니다.

```text
http://127.0.0.1:8000/health
http://127.0.0.1:8000/docs
http://127.0.0.1:8000/api/v1/edge-device/health
http://127.0.0.1:8000/edge-device
```

`/api/v1/edge-device/health`의 `auth_configured`가 `true`인지 확인합니다. **이 값은 PC에 비밀 키가 설정되었다는 뜻**이며 Pi 접속이 된 것은 아직 아닙니다.

**WSL2 네트워크 주의:** Windows 브라우저에서 `127.0.0.1`로 열린다고 해서 Pi에서도 Windows LAN IP로 자동 접속되는 것은 아닙니다. NAT·Windows 방화벽·교실의 단말 간 격리 정책에 따라 연결할 수 없습니다. 뒤에서 Pi로 직접 확인하고, 접근이 안 되면 **기관에서 허용한 WSL 미러링/포트 전달/Windows 직접 서버 실행**을 점검합니다. 확인되지 않은 외부 인터넷에 8000 포트를 공개하지 않습니다.

---

## PART 3. Raspberry Pi 카메라·GPIO·Outbox 코드를 작성합니다

실습 10~15는 **PC 교과 13 프로젝트 안에서 새 코드를 작성**합니다. 필요한 파일을 모두 만든 다음 Pi로 배포하므로, Pi에서 같은 코드를 처음부터 다시 입력하지 않습니다.

### 실습 10. GPIO LED 제어 코드를 작성합니다

이 코드는 모델의 판정을 LED 상태로 바꿉니다. 안전을 위해 두 LED를 먼저 끈 다음 해당 LED 하나만 켭니다.

**의사코드**

```text
판정 수신 → 두 LED OFF
     ├─ OK      → 파란 LED만 ON
     ├─ NG      → 빨간 LED만 ON
     └─ UNKNOWN → 모두 OFF
종료/예외 → 모두 OFF 및 GPIO 해제
```

**파일: `device/day06_pi/gpio_led.py`**

```python
"""안전한 GPIO 상태: OK=파랑, NG=빨강, UNKNOWN/예외=모두 OFF."""


class MockLED:
    def __init__(self):
        self.is_lit = False

    def on(self):
        self.is_lit = True

    def off(self):
        self.is_lit = False

    def close(self):
        self.off()


class StatusLEDs:
    def __init__(self, blue_bcm=17, red_bcm=27, dry_run=False):
        if blue_bcm == red_bcm:
            raise ValueError("두 LED에 같은 BCM 핀을 지정할 수 없습니다.")
        if dry_run:
            self.blue, self.red = MockLED(), MockLED()
        else:
            try:
                from gpiozero import LED
                self.blue, self.red = LED(blue_bcm), LED(red_bcm)
            except Exception as error:
                raise RuntimeError("gpiozero/lgpio와 GPIO 접근 권한, BCM 번호를 확인하세요.") from error
        self.set_status("UNKNOWN")

    def set_status(self, decision):
        self.blue.off()
        self.red.off()
        if decision == "OK":
            self.blue.on()
        elif decision == "NG":
            self.red.on()
        elif decision != "UNKNOWN":
            raise ValueError(f"정의하지 않은 판정: {decision}")

    def close(self):
        self.blue.off()
        self.red.off()
        self.blue.close()
        self.red.close()
```

GPIO를 연결하지 않고 실습할 때는 `dry_run=True`로 같은 상태 변경을 검사할 수 있습니다. `BCM17`, `BCM27`은 Raspberry Pi 물리 핀 번호 11, 13에 해당합니다.

### 실습 11. 전송 실패 시 결과를 보존할 Outbox를 작성합니다

Pi에서 네트워크가 끊겼다고 추론 이력을 잃지 않도록 UUID별 JSON 파일을 먼저 저장합니다. PC 응답이 정상인 경우에만 파일을 삭제합니다.

**의사코드**

```text
UUID 이벤트 생성
     ↓
outbox/{UUID}.json 파일 저장
     ↓
PC POST + X-Device-Key
     ├─ 정상 응답 → 로컬 파일 삭제
     └─ 실패 → 파일 유지, 다음 전송에서 재시도
```

**파일: `device/day06_pi/outbox.py`**

```python
"""네트워크가 끊겨도 이벤트를 JSON 파일로 보관했다가 재전송합니다."""
import json
from pathlib import Path
from urllib import request, error


class Outbox:
    def __init__(self, folder: Path):
        self.folder = Path(folder)
        self.folder.mkdir(parents=True, exist_ok=True)

    def enqueue(self, event: dict) -> Path:
        # 서버가 dedup할 UUID 문자열을 로컬 파일명으로도 사용합니다.
        from uuid import UUID
        event_id = str(UUID(event["event_id"]))
        target = self.folder / f"{event_id}.json"
        temporary = self.folder / f"{event_id}.tmp"
        temporary.write_text(json.dumps(event, ensure_ascii=False), encoding="utf-8")
        temporary.replace(target)
        return target

    def pending_count(self) -> int:
        return sum(1 for _ in self.folder.glob("*.json"))

    def flush(self, base_url: str, api_key: str, max_items: int = 8, timeout: float = 3.0) -> tuple[int, int]:
        if not base_url.startswith(("http://", "https://")):
            raise ValueError("PC 플랫폼 URL은 http(s)://로 시작해야 합니다.")
        sent = 0
        for path in sorted(self.folder.glob("*.json"))[:max_items]:
            try:
                data = path.read_bytes()
                req = request.Request(
                    base_url.rstrip("/") + "/api/v1/edge-device/events",
                    data=data, method="POST",
                    headers={"Content-Type": "application/json", "X-Device-Key": api_key},
                )
                with request.urlopen(req, timeout=timeout) as resp:
                    answer = json.loads(resp.read().decode("utf-8"))
                # 첫 저장 성공 및 동일 event_id 재전송 성공 모두 큐에서 제거합니다.
                if "saved" not in answer or "event" not in answer:
                    raise ValueError("API 응답 형식을 확인하세요.")
                path.unlink()
                sent += 1
            except (OSError, error.URLError, ValueError, json.JSONDecodeError):
                # 서버 중단/401/503 등 실패 시 이벤트 파일을 보존합니다.
                break
        return sent, self.pending_count()
```

Pi는 `outbox/`에 기록을 남기기 때문에 PC 네트워크가 일시적으로 중단되어도 데이터가 보존됩니다. 이 구현은 한 Pi에서 실행하는 단일 교육용 프로세스 기준입니다. 여러 프로세스가 하나의 Outbox 폴더를 동시에 다루도록 실행하지 않습니다.

### 실습 12. ONNX 모델 파일·설정의 무결성을 검사합니다

5일차의 `manifest.json`에는 ONNX SHA256, 클래스 순서, 입력 크기, 버전, 교육용 Confidence 기준값이 기록돼 있습니다. Pi에서 복사된 값이 같은지 확인하는 코드를 작성합니다.

```text
ONNX SHA256 계산
      ↓
Manifest의 SHA256과 비교
      ↓
[normal, warning_red] 순서 확인
      ↓
[1,3,64,64] 입력과 모델 ID/버전·Threshold 확인
      ↓
모두 일치할 때만 카메라 추론 시작
```

**파일: `device/day06_pi/verify_package.py`**

```python
"""5일차 배포물 ONNX SHA256·클래스·전처리 설정을 검사합니다."""
import hashlib
import json
from pathlib import Path


def verify(folder: Path) -> dict:
    folder = Path(folder)
    manifest = json.loads((folder / "manifest.json").read_text(encoding="utf-8"))
    config = json.loads((folder / "training_config.json").read_text(encoding="utf-8"))
    model = folder / "tinycnn.onnx"
    digest = hashlib.sha256(model.read_bytes()).hexdigest()
    if digest != manifest["onnx_sha256"]:
        raise ValueError("ONNX SHA256이 5일차 Manifest와 일치하지 않습니다.")
    if config["classes"] != manifest["classes"] or config["classes"] != ["normal", "warning_red"]:
        raise ValueError("클래스 순서가 일치하지 않습니다.")
    if manifest["input_shape"] != [1, 3, config["image_size"], config["image_size"]]:
        raise ValueError("ONNX 입력 크기가 일치하지 않습니다.")
    if config["model_id"] != manifest["model_id"] or config["model_version"] != manifest["version"]:
        raise ValueError("모델 이름/버전이 일치하지 않습니다.")
    if abs(float(config["confidence_threshold_demo"]) - float(manifest["threshold_demo"])) > 1e-8:
        raise ValueError("UNKNOWN 신뢰도 기준이 일치하지 않습니다.")
    return {"model_id": config["model_id"], "version": config["model_version"], "sha256": digest}


if __name__ == "__main__":
    print("모델 검증 PASS:", verify(Path(__file__).resolve().parent))
```

SHA256이 일치하지 않으면 무작정 해시를 수정하지 않고, **5일차에 실제 검증한 원본 배포 파일**을 다시 복사합니다.

### 실습 13. Raspberry Pi 카메라 → ONNX → GPIO → 전송을 연결합니다

이 파일은 6일차의 **핵심 실행 코드**입니다. 5일차 `runtime.py`에 이미 있는 `predict_image_bytes()`를 그대로 import합니다. 따라서 ONNX 전처리를 다시 구현하지 않습니다.

**의사코드**

```text
설정 로드 → ONNX Manifest 검사
       ↓
두 LED OFF / Outbox 준비 / 송신 스레드 시작
       ↓
USB 카메라 연결 시도
       ├─ 실패 → CAMERA_UNAVAILABLE + OFF + 재시도
       └─ 성공 → Frame 읽기
                     ↓
           5일차 ONNX CPU 추론
                     ↓
            OK / NG / UNKNOWN
                     ↓
              LED 상태 변경
                     ↓
   판정이 바뀌거나 기록 시간이 경과하면 UUID 이벤트 저장
                     ↓
        별도 송신 스레드가 PC로 POST
       ↓
Ctrl+C → 카메라 해제 / LED OFF / Outbox 유지
```

**파일: `device/day06_pi/run_camera.py`**

```python
"""Raspberry Pi USB 카메라 → ONNX CPU → GPIO → 로컬 Outbox → PC API."""
import argparse
import json
import os
import threading
import time
from datetime import datetime, timezone
from pathlib import Path
from uuid import uuid4

import cv2

from gpio_led import StatusLEDs
from outbox import Outbox
from runtime import predict_image_bytes
from verify_package import verify

HERE = Path(__file__).resolve().parent


def build_event(config: dict, model_info: dict, result: dict | None, reason: str | None = None) -> dict:
    if result is None:
        if reason not in ("CAMERA_UNAVAILABLE", "INFERENCE_ERROR"):
            raise ValueError("실패 이벤트의 reason_code가 필요합니다.")
        label, confidence, decision, duration = None, None, "UNKNOWN", None
        reason_code = reason
    else:
        label, confidence = result["predicted_label"], result["confidence"]
        decision, duration = result["decision"], result["inference_ms"]
        reason_code = ("LOW_CONFIDENCE" if decision == "UNKNOWN" else
                       "NORMAL" if decision == "OK" else "WARNING_RED")
    return {
        "event_id": str(uuid4()),
        "device_id": config["device_id"],
        "captured_at": datetime.now(timezone.utc).isoformat(),
        "input_ref": "usb-camera-0" if isinstance(config["camera_source"], int) else "ip-camera",
        "predicted_label": label, "confidence": confidence,
        "decision": decision, "reason_code": reason_code, "inference_ms": duration,
        "model_id": model_info["model_id"],
        "model_version": model_info["version"],
        "model_sha256": model_info["sha256"],
    }


def sender_loop(outbox: Outbox, stop: threading.Event, config: dict, api_key: str):
    while not stop.is_set():
        try:
            sent, pending = outbox.flush(config["api_base"], api_key)
            if sent:
                print(f"[전송] 성공 {sent}건, 대기 {pending}건", flush=True)
            elif pending:
                print(f"[전송 대기] 네트워크/서버를 확인하세요. 보관 {pending}건", flush=True)
        except Exception as error:
            # 시크릿 키, 요청 본문은 출력하지 않습니다.
            print(f"[전송 오류] {type(error).__name__}", flush=True)
        stop.wait(config["send_interval_sec"])


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--config", type=Path, default=HERE / "device_config.json")
    parser.add_argument("--dry-run", action="store_true", help="LED를 실제 GPIO에 연결하지 않습니다.")
    parser.add_argument("--image", type=Path, help="카메라 없이 이미지 한 장으로 추론·전송 큐만 확인")
    args = parser.parse_args()
    config = json.loads(args.config.expanduser().read_text(encoding="utf-8"))
    for key in ("device_id", "api_base", "camera_source", "capture_interval_sec", "event_interval_sec", "send_interval_sec"):
        if key not in config:
            raise ValueError(f"설정 파일에 {key}가 없습니다.")
    for key in ("capture_interval_sec", "event_interval_sec", "send_interval_sec"):
        if not 0.1 <= float(config[key]) <= 120:
            raise ValueError(f"{key}는 0.1~120초로 설정합니다.")
    model_info = verify(HERE)   # SHA256 및 전처리 설정의 원본 동일성 확인
    model = HERE / "tinycnn.onnx"
    training = HERE / "training_config.json"
    outbox = Outbox(HERE / "outbox")
    api_key = os.getenv("DAY06_DEVICE_API_KEY", "")
    if len(api_key) < 24:
        raise RuntimeError("DAY06_DEVICE_API_KEY 환경변수를 설정하세요.")
    leds = StatusLEDs(config.get("blue_bcm", 17), config.get("red_bcm", 27),
                      dry_run=args.dry_run or not config.get("use_gpio", True))
    stop = threading.Event()
    sender = None  # 이미지 1회 점검에서는 백그라운드 전송을 시작하지 않습니다.
    capture = None
    last_event_time = -1e10
    previous_decision = None
    last_error_kind = None
    try:
        if args.image:
            # 카메라와 GPIO가 없는 상태에서 5일차 이미지로 전체 데이터를 확인합니다.
            result = predict_image_bytes(args.image.read_bytes(), model, training)
            leds.set_status(result["decision"])
            event = build_event(config, model_info, result)
            outbox.enqueue(event)
            print("[이미지 1회 테스트]", json.dumps(event, ensure_ascii=False, indent=2))
            sent, pending = outbox.flush(config["api_base"], api_key)
            print(f"[전송 확인] {sent}건 성공, {pending}건 보관")
            return
        sender = threading.Thread(target=sender_loop, args=(outbox, stop, config, api_key), daemon=True)
        sender.start()
        while not stop.is_set():
            if capture is None or not capture.isOpened():
                capture = cv2.VideoCapture(config["camera_source"])
                if not capture.isOpened():
                    leds.set_status("UNKNOWN")
                    if last_error_kind != "CAMERA_UNAVAILABLE":
                        outbox.enqueue(build_event(config, model_info, None, "CAMERA_UNAVAILABLE"))
                        last_error_kind = "CAMERA_UNAVAILABLE"
                    print("[카메라 없음] 연결을 확인하고 3초 후 다시 시도합니다.", flush=True)
                    capture.release()
                    capture = None
                    stop.wait(3)
                    continue
                last_error_kind = None
                print("[카메라 연결 성공]", flush=True)
            ok, frame = capture.read()
            if not ok or frame is None:
                leds.set_status("UNKNOWN")
                if last_error_kind != "CAMERA_UNAVAILABLE":
                    outbox.enqueue(build_event(config, model_info, None, "CAMERA_UNAVAILABLE"))
                    last_error_kind = "CAMERA_UNAVAILABLE"
                capture.release()
                capture = None
                stop.wait(2)
                continue
            try:
                ok_jpg, data = cv2.imencode(".jpg", frame)
                if not ok_jpg:
                    raise ValueError("카메라 프레임 JPEG 인코딩 실패")
                result = predict_image_bytes(data.tobytes(), model, training)
                decision = result["decision"]
                leds.set_status(decision)
                last_error_kind = None
                print(f"[예측] {result['predicted_label']} / {decision} / "
                      f"Confidence {result['confidence']:.3f} / ONNX {result['inference_ms']}ms", flush=True)
            except Exception as error:
                leds.set_status("UNKNOWN")
                decision = "UNKNOWN"
                result = None
                print(f"[추론 오류] {type(error).__name__}: 전처리/ONNX를 점검하세요.", flush=True)
            now = time.monotonic()
            if decision != previous_decision or now - last_event_time >= config["event_interval_sec"]:
                outbox.enqueue(build_event(config, model_info, result,
                                           "INFERENCE_ERROR" if result is None else None))
                previous_decision = decision
                last_event_time = now
            stop.wait(config["capture_interval_sec"])
    except KeyboardInterrupt:
        print("\n[종료] GPIO를 끄고 송신 작업을 정리합니다.")
    finally:
        stop.set()
        if capture is not None:
            capture.release()
        leds.close()
        if sender is not None:
            sender.join(timeout=4)
        print("[종료 완료] 전송되지 않은 파일은 outbox/에 남아 있습니다.")


if __name__ == "__main__":
    main()
```

**코드 이해하기**

`build_event()`는 UUID·UTC 시간·장치 ID·모델 해시를 결과에 추가합니다. `sender_loop()`는 별도 스레드에서 파일을 전송하므로, 네트워크 지연을 `inference_ms`로 오해하지 않습니다. 연결이 끊긴 카메라에 대해서는 예측 라벨과 Confidence를 `null`로 기록합니다.

`--image` 옵션은 카메라를 연결하지 않고 정상·빨간 카드 Test 이미지 한 장으로 ONNX→Outbox→HTTP 전체 흐름을 확인합니다. `--dry-run` 옵션은 실제 GPIO 대신 가상 LED를 사용하지만 **실제 ONNX 추론은 그대로 실행합니다.**

### 실습 14. 장치 설정 파일을 작성합니다

`device_id`, 카메라 번호와 PC 주소를 장치별로 변경할 수 있도록 설정을 분리합니다. 비밀 키는 이 파일에 넣지 않습니다.

**파일: `device/day06_pi/device_config.example.json`**

```json
{
  "device_id": "pi-student-01",
  "api_base": "http://192.168.0.20:8000",
  "camera_source": 0,
  "blue_bcm": 17,
  "red_bcm": 27,
  "capture_interval_sec": 0.5,
  "event_interval_sec": 3.0,
  "send_interval_sec": 2.0,
  "use_gpio": true
}
```

`api_base`에 적힌 IP는 **예시 값**입니다. 실제 Pi에서 접근 가능한 PC의 LAN IPv4로 바꾸어야 합니다. `camera_source: 0`은 첫 USB 카메라를 가리킵니다. `capture_interval_sec`과 `event_interval_sec`은 캡처 간격 및 같은 판정의 기록 간격입니다.

Pi의 ONNX Runtime만 pip로 별도 설치하는 패키지 목록을 작성합니다. OpenCV·NumPy·Pillow·gpiozero·lgpio는 Pi OS apt 패키지를 우선 이용합니다.

**파일: `device/day06_pi/requirements_pi.txt`**

```text
# Raspberry Pi OS ARM64: OpenCV, NumPy, Pillow, gpiozero, lgpio는 apt 설치를 우선 사용합니다.
# ONNX Runtime의 현재 Python/ARM64 호환 wheel 존재를 반드시 사전 확인합니다.
onnxruntime>=1.18,<2
```

### 실습 15. 5일차 배포 파일과 6일차 코드를 한 폴더로 묶습니다

`deploy/day05_raspberrypi/`의 ONNX·설정·런타임 파일을 **수정하지 않고 복사**합니다. 6일차에 추가한 코드까지 하나의 폴더로 모으는 패키징 스크립트를 작성합니다.

**의사코드**

```text
5일차 배포 파일 5개 확인
      +
6일차 장치 파일 6개 확인
      ↓
deploy/day06_raspberrypi/ 생성·복사
      ↓
SHA256·클래스·모델 설정 검사
      ↓
Pi로 전송 가능한 패키지 완성
```

**파일: `scripts/16_package_day06.py`**

```python
"""5일차 Raspberry Pi 파일 + 6일차 GPIO·큐·카메라 코드를 함께 배포."""
import shutil
import json
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]
PREVIOUS = ROOT / "deploy" / "day05_raspberrypi"
DEVICE = ROOT / "device" / "day06_pi"
TARGET = ROOT / "deploy" / "day06_raspberrypi"


def main():
    previous_files = ["tinycnn.onnx", "training_config.json", "manifest.json", "runtime.py", "infer_on_device.py"]
    next_files = ["gpio_led.py", "outbox.py", "run_camera.py", "verify_package.py",
                  "device_config.example.json", "requirements_pi.txt"]
    for folder, filenames in ((PREVIOUS, previous_files), (DEVICE, next_files)):
        for name in filenames:
            if not (folder / name).is_file():
                raise FileNotFoundError(f"6일차 필수 파일 누락: {folder / name}")
    TARGET.mkdir(parents=True, exist_ok=True)
    for folder, filenames in ((PREVIOUS, previous_files), (DEVICE, next_files)):
        for name in filenames:
            shutil.copy2(folder / name, TARGET / name)
    from device.day06_pi.verify_package import verify
    info = verify(TARGET)
    print("6일차 Pi 배포 폴더 준비 완료:", TARGET)
    print("5일차 모델 ID / 버전:", info["model_id"], info["version"])
    print("ONNX SHA256:", info["sha256"])
    for file in sorted(TARGET.iterdir()):
        print(" -", file.name)


if __name__ == "__main__":
    main()
```

PC의 교과 13 프로젝트에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m scripts.16_package_day06
find deploy/day06_raspberrypi -maxdepth 1 -type f | sort
```

정상적인 구조는 다음과 같습니다.

```text
deploy/day06_raspberrypi/
├── tinycnn.onnx                   # 5일차 모델
├── training_config.json          # 5일차 설정
├── manifest.json                 # 5일차 SHA256
├── runtime.py                    # 5일차 CPU 전처리·추론
├── infer_on_device.py            # 5일차 1장 이미지 CLI
├── gpio_led.py                   # 6일차 LED
├── outbox.py                     # 6일차 재전송 대기함
├── verify_package.py             # 6일차 SHA256 검사
├── run_camera.py                 # 6일차 카메라 통합
├── device_config.example.json    # 장치별 수정 예시
└── requirements_pi.txt           # Pi용 ONNX Runtime
```

`tinycnn.onnx`의 해시가 Manifest와 다르면 다음 단계로 진행하지 않습니다.

---

## PART 4. Raspberry Pi에 배포하고 CPU 추론을 검증합니다

### 실습 16. PC에서 Pi로 파일을 전송합니다

Raspberry Pi에 SSH로 접속할 수 있어야 합니다. 아래 `student@192.168.0.30`은 예시이며 실제 장치의 계정·주소로 바꾸세요. **PC WSL 터미널에서 실행**합니다.

```bash
ssh student@192.168.0.30 'mkdir -p ~/subject13_edge'
scp -r deploy/day06_raspberrypi student@192.168.0.30:~/subject13_edge/
```

정상·빨간 카드 Test 이미지도 각 한 장씩 전달합니다. 교과 5일차 실습에서 직접 촬영한 파일명이 아래와 다르면 존재하는 파일 이름으로 수정합니다.

```bash
ssh student@192.168.0.30 'mkdir -p ~/subject13_edge/test_images'
scp data/day05/images/test/normal/normal_test_001.jpg \
  student@192.168.0.30:~/subject13_edge/test_images/normal.jpg
scp data/day05/images/test/warning_red/warning_red_test_001.jpg \
  student@192.168.0.30:~/subject13_edge/test_images/warning_red.jpg
```

**6일차 패키지는 ONNX 모델을 포함하므로 GitHub에 올리지 않습니다.** 승인된 장치 간 SSH·외장 저장장치를 이용하여 옮깁니다.

### 실습 17. Pi OS에서 필요한 Python 패키지를 설치합니다

**지금부터 Pi SSH 터미널**에서 실행합니다.

```bash
uname -m
python3 --version
cat /etc/os-release | head -6
sudo apt update
sudo apt install -y python3-venv python3-opencv python3-numpy python3-pil \
  python3-gpiozero python3-lgpio
cd ~/subject13_edge/day06_raspberrypi
python3 -m venv --system-site-packages .venv
source .venv/bin/activate
python - <<'PY'
import cv2, numpy, PIL, gpiozero
print('OpenCV:', cv2.__version__)
print('NumPy:', numpy.__version__)
print('Pillow:', PIL.__version__)
print('GPIO Zero: import OK')
PY
```

OS 패키지명은 Raspberry Pi OS 버전에 따라 다를 수 있습니다. `aarch64`와 현재 `python3 --version`을 확인한 뒤 **현재 Pi OS와 호환되는 ARM64 패키지**를 사용합니다. 개발 PC의 `.venv`나 x86_64 Wheel을 복사하지 않습니다.

ONNX Runtime은 해당 Python 버전의 ARM64 Wheel을 설치합니다. 빌드를 오래 시작하지 않도록 Wheel 설치로 제한합니다.

```bash
python -m pip install --only-binary=:all: -r requirements_pi.txt
python - <<'PY'
import onnxruntime as ort
print('ONNX Runtime:', ort.__version__)
print('Providers:', ort.get_available_providers())
assert 'CPUExecutionProvider' in ort.get_available_providers()
PY
```

인터넷이 차단된 교육장에서는 승인된 ARM64 Wheel 저장소를 이용합니다. 설치 실패 시에는 Pi OS, Python 버전, Wheel 지원 여부부터 확인하고 다음 단계로 넘어가지 않습니다.

### 실습 18. Pi에서 5일차 ONNX 파일을 검증합니다

**Pi 터미널**에서 실행합니다.

```bash
cd ~/subject13_edge/day06_raspberrypi
source .venv/bin/activate
python verify_package.py
cat training_config.json
```

`모델 검증 PASS`가 출력되면 5일차 SHA256과 클래스·입력 크기·버전·Threshold가 일치합니다.

### 실습 19. 카메라 없이 Test 사진 한 장으로 먼저 추론합니다

5일차에 만든 `infer_on_device.py`를 그대로 이용합니다. GPU나 PyTorch를 Pi에 설치하지 않습니다.

```bash
python infer_on_device.py --image ~/subject13_edge/test_images/normal.jpg
python infer_on_device.py --image ~/subject13_edge/test_images/warning_red.jpg
```

다음과 같은 **키들이 출력되는지** 확인합니다.

```json
{
  "predicted_label": "normal",
  "confidence": 0.9,
  "decision": "OK",
  "inference_ms": 12.3,
  "model_id": "tinycnn-day05",
  "model_version": "day05-1.0",
  "source_mode": "edge_training_demo"
}
```

예시 숫자이며 실제 모델과 Pi 속도에 따라 달라집니다. 실제 클래스 예측이 틀릴 수 있으므로 결과가 예시와 반드시 같아야 하는 것은 아닙니다. 먼저 **모델 로딩과 실제 추론**이 오류 없이 성공하는지 확인합니다.

---

## PART 5. GPIO와 PC PostgreSQL 전송을 연결합니다

### 실습 20. Pi 장치 설정과 키를 준비합니다

Pi 배포 폴더에서 설정 파일을 복사합니다.

```bash
cd ~/subject13_edge/day06_raspberrypi
cp device_config.example.json device_config.json
nano device_config.json
```

`api_base`는 **Pi에서 접근 가능한 PC의 실제 LAN IPv4**, `device_id`는 다른 장치와 중복되지 않는 값으로 변경합니다. 카메라 번호·BCM 핀도 실제 배선에 맞춥니다.

비밀 키는 Pi의 **별도 `.env` 파일**에 저장합니다.

```bash
nano .env
```

```dotenv
DAY06_DEVICE_API_KEY=PC와_정확히_동일한_비밀키
```

```bash
chmod 600 .env
set -a
source .env
set +a
python - <<'PY'
import os
assert len(os.getenv('DAY06_DEVICE_API_KEY', '')) >= 24
print('Pi 장치 키 준비 완료')
PY
```

키 자체를 터미널에 출력하거나 공개하지 않습니다. 장치 `.env`, `device_config.json`, `outbox/`는 원격 Git 대상이 아닙니다.

### 실습 21. Pi에서 PC API 접근을 확인합니다

**Windows PowerShell**에서 PC의 유선/Wi-Fi IPv4를 확인합니다.

```powershell
ipconfig
```

예시 `192.168.0.20`을 PC 실제 IPv4로 바꾼 뒤 **Pi 터미널**에서 호출합니다.

```bash
curl --max-time 5 http://192.168.0.20:8000/api/v1/edge-device/health
```

`service: ready`, `auth_configured: true`가 나타나면 Pi가 PC의 API에 접근하는 데 성공했습니다. **Pi에서 `127.0.0.1:8000`을 호출하면 Pi 자신을 확인하는 것이므로 사용하지 않습니다.**

접근이 실패하면 PC FastAPI 실행·Windows 방화벽·WSL2 NAT/미러링·Wi-Fi 단말 격리 등을 차례대로 확인합니다. 임의의 외부 IP에 비밀 키를 전송하지 않습니다.

### 실습 22. 카메라·GPIO 없이 Pi→PC 전송부터 확인합니다

**Pi 터미널**에서 실제 ONNX를 실행하되 GPIO만 가상 LED로 대체합니다.

```bash
cd ~/subject13_edge/day06_raspberrypi
source .venv/bin/activate
set -a && source .env && set +a
python run_camera.py --dry-run --image ~/subject13_edge/test_images/normal.jpg
```

`event_id`, `captured_at`, `model_sha256`, `decision`을 출력하고 `전송 확인` 결과를 보여야 합니다. PC API가 정상 응답했다면 `1건 성공, 0건 보관`이 나옵니다. PC에서 다음 주소를 엽니다.

```text
http://127.0.0.1:8000/api/v1/edge-device/events
http://127.0.0.1:8000/edge-device
```

Pi 결과가 1건 이상 보이는지 확인합니다. **같은 사진을 다시 실행하는 것은 UUID가 새로 생성되어 새 이벤트가 추가**됩니다. 같은 UUID를 네트워크 때문에 재전송한 경우에만 중복 저장하지 않습니다.

### 실습 23. 전원을 끄고 LED를 배선한 뒤 상태를 검사합니다

Pi 전원을 끈 상태에서 **BCM17(물리 11)→330Ω→파란 LED→GND**, **BCM27(물리 13)→330Ω→빨간 LED→GND**를 연결합니다. GPIO에 5V를 직접 연결하거나 저항 없이 LED를 연결하지 않습니다.

다시 켜서 SSH로 접속한 후 **Pi 터미널**에서 실행합니다.

```bash
cd ~/subject13_edge/day06_raspberrypi
source .venv/bin/activate
export GPIOZERO_PIN_FACTORY=lgpio
python - <<'PY'
from gpio_led import StatusLEDs
import time
leds = StatusLEDs(17,27,dry_run=False)
try:
    print('파랑'); leds.set_status('OK'); time.sleep(2)
    print('빨강'); leds.set_status('NG'); time.sleep(2)
    print('모두 OFF'); leds.set_status('UNKNOWN'); time.sleep(2)
finally:
    leds.close()
PY
```

LED가 켜지지 않으면 BCM 번호·저항·극성·접지·GPIO 접근 권한을 확인합니다. `sudo python`으로 무조건 실행하거나 GPIO 디바이스 권한을 전체 공개하는 방식으로 해결하지 않습니다.

### 실습 24. USB 카메라 연속 추론과 DB 저장을 실행합니다

USB UVC 카메라를 Raspberry Pi에 연결합니다. SSH 환경이므로 `cv2.imshow()`를 호출하지 않습니다. **Pi 터미널**에서 다음을 실행합니다.

```bash
cd ~/subject13_edge/day06_raspberrypi
source .venv/bin/activate
set -a && source .env && set +a
export GPIOZERO_PIN_FACTORY=lgpio
python run_camera.py --config device_config.json
```

5일차와 같은 종류의 정상 카드와 빨간 카드가 각각 입력될 때 예측이 어떻게 바뀌는지 확인합니다.

```text
예측 normal / OK → 파란 LED ON
예측 warning_red / NG → 빨간 LED ON
Confidence 낮음 / UNKNOWN → LED 모두 OFF
카메라 오류 / UNKNOWN → LED 모두 OFF
```

**실제 예측값은 모델 성능에 따라 다릅니다.** 빨간 카드를 항상 인식한다고 보장하지 않습니다. 웹 화면에서 Pi 장치 ID·Confidence·기록 시각·ONNX 추론시간이 실제로 저장되는지 확인합니다.

Pi 터미널에서 `Ctrl+C`를 누르면 `finally`에서 GPIO를 끄고 카메라를 해제합니다.

### 실습 25. 연결 오류를 확인하고 복구합니다

**PC 서버 중지 시험:** Pi에서 `run_camera.py`가 실행 중일 때 PC Uvicorn을 잠시 중지합니다. Pi의 추론과 LED는 계속되지만 HTTP 전송이 실패해 `outbox/`에 JSON이 쌓입니다. **다른 Pi 터미널**에서 확인합니다.

```bash
cd ~/subject13_edge/day06_raspberrypi
find outbox -maxdepth 1 -type f -name '*.json' | wc -l
```

PC Uvicorn을 다시 시작하면 대기 파일이 재전송됩니다. 정상 응답 후 outbox의 파일이 줄어드는지 확인합니다.

**USB 카메라 중단 시험:** 카메라가 끊기면 `CAMERA_UNAVAILABLE` 이벤트와 `UNKNOWN`을 기록하고 LED를 모두 끈 뒤 재연결을 시도합니다. 실제 장비의 USB 분리 지침을 지키고, 재연결 후 추론이 복구되는지 확인합니다.

**키 불일치:** 장치 키가 틀리면 PC API는 401을 반환합니다. Outbox는 기록을 삭제하지 않으며, 키를 교정하고 재실행하면 저장할 수 있습니다.

**주의:** 이 로컬 Outbox는 재전송 유실을 줄이지만 디스크 자체의 고장·동시 프로세스 접근·무한 데이터 증가까지 해결하는 상용 보존 시스템은 아닙니다. 교과 13의 다음 운영·모니터링 단계에서 추가 관리합니다.

---

## PART 6. 자동 테스트·PostgreSQL 조회·운영 문서를 작성합니다

### 실습 26. PC에서 6일차 자동 테스트를 작성합니다

새 테스트 파일은 GPIO 실제 회로나 실제 ONNX Runtime 추론 속도를 검증하는 것이 아니라, **인증·이벤트 중복 방지·판정 규칙·GPIO 가상 상태·Outbox·Manifest**를 검사합니다.

**검증 흐름**

```text
키 없음/틀림 → 401
   ↓
정상 이벤트 → DB 저장
   ↓
같은 UUID 재전송 → 중복 방지
   ↓
카메라 오류 → UNKNOWN + Confidence NULL
   ↓
잘못된 판정·예측 조합 → 422
   ↓
가상 GPIO → OK/NG/UNKNOWN 상태 검사
   ↓
네트워크 오류 → Outbox 파일 보존
   ↓
전송 성공 → Outbox 파일 삭제
   ↓
SHA256 변경 → 모델 검증 실패
```

**파일: `tests/test_day06.py`**

```python
"""6일차: 장치 인증·DB 저장/중복 방지·GPIO·재전송·Manifest 검증."""
import hashlib
import json
from datetime import datetime, timezone
from pathlib import Path
from uuid import uuid4

import pytest
from fastapi import FastAPI
from fastapi.testclient import TestClient

from app.db import Base, engine, SessionLocal, get_db
from app import models  # noqa: F401 - ORM 테이블 등록
from app.edge_device_api import router
from device.day06_pi.gpio_led import StatusLEDs
from device.day06_pi.outbox import Outbox
from device.day06_pi.verify_package import verify

KEY = "classroom-device-test-key-very-long-123"


def payload(**updates):
    base = {
        "event_id": str(uuid4()), "device_id": "pi-student-01",
        "captured_at": datetime.now(timezone.utc).isoformat(),
        "input_ref": "usb-camera-0", "predicted_label": "normal",
        "confidence": 0.91, "decision": "OK", "reason_code": "NORMAL",
        "inference_ms": 12.3, "model_id": "tinycnn-day05", "model_version": "day05-1.0",
        "model_sha256": "a"*64,
    }
    base.update(updates)
    return base


@pytest.fixture
def client(tmp_path, monkeypatch):
    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker
    db_file = tmp_path / "day06.sqlite3"
    local_engine = create_engine(f"sqlite:///{db_file}")
    Base.metadata.create_all(local_engine)
    local_session = sessionmaker(bind=local_engine, expire_on_commit=False)
    app = FastAPI()
    app.include_router(router)
    def override_db():
        with local_session() as session:
            yield session
    app.dependency_overrides[get_db] = override_db
    monkeypatch.setenv("DAY06_DEVICE_API_KEY", KEY)
    with TestClient(app) as c:
        yield c
    local_engine.dispose()


def test_health_without_secret_exposure(client):
    response = client.get("/api/v1/edge-device/health")
    assert response.status_code == 200
    assert response.json()["auth_configured"] is True
    assert KEY not in response.text


def test_reject_missing_key(client):
    response = client.post("/api/v1/edge-device/events", json=payload())
    assert response.status_code == 401


def test_reject_wrong_key(client):
    response = client.post("/api/v1/edge-device/events", json=payload(), headers={"X-Device-Key": "wrong"})
    assert response.status_code == 401


def test_pi_ok_insert_deduplicate_and_list(client):
    item = payload()
    headers = {"X-Device-Key": KEY}
    first = client.post("/api/v1/edge-device/events", json=item, headers=headers)
    assert first.status_code == 200, first.text
    assert first.json()["saved"] is True
    event = first.json()["event"]
    assert event["event_id"] == "pi:" + item["event_id"]
    assert event["source_mode"] == "live_inference"
    assert event["details"]["device_kind"] == "raspberry_pi"
    second = client.post("/api/v1/edge-device/events", json=item, headers=headers)
    assert second.status_code == 200
    assert second.json()["saved"] is False
    records = client.get("/api/v1/edge-device/events").json()
    assert len(records) == 1
    assert records[0]["decision"] == "OK"


def test_pi_unknown_camera_disconnected(client):
    item = payload(predicted_label=None, confidence=None, inference_ms=None,
                   decision="UNKNOWN", reason_code="CAMERA_UNAVAILABLE")
    response = client.post("/api/v1/edge-device/events", json=item,
                           headers={"X-Device-Key": KEY})
    assert response.status_code == 200
    event = response.json()["event"]
    assert event["decision"] == "UNKNOWN"
    assert event["confidence"] is None
    assert event["reasons"] == ["CAMERA_UNAVAILABLE"]


def test_reject_contradictory_decision(client):
    response = client.post("/api/v1/edge-device/events", json=payload(decision="OK", predicted_label="warning_red"),
                           headers={"X-Device-Key": KEY})
    assert response.status_code == 422


def test_gpio_off_for_unknown_and_safe_close():
    leds = StatusLEDs(dry_run=True)
    assert not leds.blue.is_lit and not leds.red.is_lit
    leds.set_status("OK")
    assert leds.blue.is_lit and not leds.red.is_lit
    leds.set_status("NG")
    assert leds.red.is_lit and not leds.blue.is_lit
    leds.set_status("UNKNOWN")
    assert not leds.blue.is_lit and not leds.red.is_lit
    leds.close()
    assert not leds.blue.is_lit and not leds.red.is_lit


def test_outbox_persists_on_connection_failure(tmp_path):
    box = Outbox(tmp_path / "outbox")
    item = payload()
    box.enqueue(item)
    sent, pending = box.flush("http://127.0.0.1:1", KEY, timeout=0.1)
    assert sent == 0 and pending == 1
    assert json.loads(next((tmp_path / "outbox").glob("*.json")).read_text())["event_id"] == item["event_id"]


def test_outbox_removes_file_only_after_server_ack(tmp_path, monkeypatch):
    import device.day06_pi.outbox as queue_mod
    box = Outbox(tmp_path / "queue")
    box.enqueue(payload())
    class Response:
        def __enter__(self): return self
        def __exit__(self, *args): return False
        def read(self): return b'{"saved": false, "event": {"event_id": "pi:test"}}'
    monkeypatch.setattr(queue_mod.request, "urlopen", lambda req, timeout: Response())
    sent, pending = box.flush("http://localhost:8000", KEY)
    assert (sent, pending) == (1, 0)


def test_verify_day05_model_hash_and_classes(tmp_path):
    model = b"fake ONNX bytes for manifest integrity check"
    (tmp_path / "tinycnn.onnx").write_bytes(model)
    config = {"classes": ["normal", "warning_red"], "image_size": 64,
              "model_id": "tinycnn-day05", "model_version": "day05-1.0",
              "confidence_threshold_demo": 0.6}
    manifest = {"onnx_sha256": hashlib.sha256(model).hexdigest(),
                "classes": config["classes"], "input_shape": [1,3,64,64],
                "model_id": "tinycnn-day05", "version": "day05-1.0", "threshold_demo": 0.6}
    (tmp_path / "training_config.json").write_text(json.dumps(config))
    (tmp_path / "manifest.json").write_text(json.dumps(manifest))
    assert verify(tmp_path)["sha256"] == manifest["onnx_sha256"]
    (tmp_path / "tinycnn.onnx").write_bytes(b"modified model")
    with pytest.raises(ValueError, match="SHA256"):
        verify(tmp_path)
```

테스트를 작성한 뒤 PC 프로젝트 최상위에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m pytest -q tests/test_day06.py
python -m pytest -q
```

6일차 신규 테스트가 정상이라면 다음처럼 나타납니다.

```text
..........                                                     [100%]
10 passed
```

**검증 범위:** SQLite 임시 DB, 가상 LED 및 모의 HTTP 응답을 사용한 6일차 코드를 검사했습니다. 실제 PostgreSQL·Pi ARM64 ONNX Runtime·카메라·GPIO 통합 실행은 위의 장치 실습에서 확인해야 합니다. 전체 `pytest` 실행은 학생이 1~5일차 실제 코드를 모두 가진 PC에서 수행합니다.

### 실습 27. PostgreSQL에서 Pi 기록을 직접 조회합니다

PC에서 2일차 PostgreSQL의 실제 계정·DB 이름을 사용합니다.

```bash
psql -h 127.0.0.1 -U subject13_user -d subject13_platform
```

다음 SQL로 6일차 Pi 기록만 확인합니다.

```sql
SELECT e.id, e.event_id, e.task, e.source_mode,
       e.predicted_label, e.decision,
       s.source_key, e.recorded_at_raw
FROM inference_events AS e
JOIN camera_sources AS s ON e.source_id = s.id
WHERE s.source_key LIKE 'day06-pi-%'
ORDER BY e.id DESC
LIMIT 20;
```

`source_key`가 `day06-pi-장치ID`이고 `source_mode`가 `live_inference`인지 확인합니다. 교과 9·10 기록이나 5일차 모델을 삭제하지 않습니다.

### 실습 28. 운영 메모·README·구조도를 작성합니다

다음 파일은 현재 실행 명령과 복구 방법을 다시 찾기 위한 문서입니다. 학생들은 실제 Pi 주소와 GPIO 번호, 실행 후 관찰한 문제를 추가로 기록할 수 있습니다. **실제 비밀 키·개인 사진은 적지 않습니다.**

**파일: `docs/day06_pi_runbook.md`**

````markdown
# 교과 13 · 6일차 Raspberry Pi 운영 메모

## 장치 구조
Pi USB Camera → TinyCNN ONNX CPU → GPIO → Outbox → PC FastAPI → PostgreSQL

## 실행 전 확인
- Pi `uname -m` = aarch64 / ONNX Runtime `CPUExecutionProvider`
- `python verify_package.py` : Manifest·SHA256 일치
- `device_config.json` : PC IP·장치 ID·카메라·BCM 번호 확인
- PC `.env` / Pi `.env` : 장치 인증 키 일치
- 파란 LED BCM17·빨간 LED BCM27 각각 330Ω 저항 연결

## Raspberry Pi 실행
```bash
cd ~/subject13_edge/day06_raspberrypi
source .venv/bin/activate
set -a && source .env && set +a
export GPIOZERO_PIN_FACTORY=lgpio
python run_camera.py --config device_config.json
```

## 종료·복구
- Ctrl+C : GPIO OFF, 카메라 해제
- CAMERA_UNAVAILABLE : USB·카메라 번호·권한 확인
- INFERENCE_ERROR : ONNX Runtime·전처리·파일 SHA256 확인
- 인증 401 : PC/Pi 키 일치 확인(실제 키 출력 금지)
- HTTP 연결 실패 : PC LAN IP·방화벽·WSL2 확인
- Outbox 누적 : PC 복구 후 자동 재전송, 남은 파일 확인

## PC 조회
- `/api/v1/edge-device/health`
- `/api/v1/edge-device/events`
- `/edge-device`

## 지표 해석
- inference_ms : Pi ONNX 추론시간, 카메라·전송·DB 지연은 포함 안 됨
- normal/warning_red : 교육용 두 카드 클래스, 실제 현업 불량·안전 판정 아님
````

**`README.md` 맨 아래에 추가**

````markdown
## 6일차 — Raspberry Pi ONNX·GPIO·PC 결과 저장

- 5일차 ONNX 원본: `deploy/day05_raspberrypi/`
- 6일차 패키지: `python -m scripts.16_package_day06`
- Pi 실행: `python run_camera.py --config device_config.json`
- PC 조회: `http://127.0.0.1:8000/edge-device`
- PC 저장: 2일차 PostgreSQL `inference_events` 재사용
- 장애 복구: Pi `outbox/` 보관·재전송
- 신규 테스트: `python -m pytest -q tests/test_day06.py`
````

**`docs/architecture.md` 맨 아래에 추가**

````markdown
## 6일차 — Raspberry Pi 장치 연결

5일차 tinycnn.onnx + training_config.json + manifest.json
          ↓
Raspberry Pi USB Camera → ONNX Runtime CPU → GPIO LED
          ↓
Pi Outbox JSON → 인증된 HTTP POST
          ↓
PC FastAPI /api/v1/edge-device/events
          ↓
기존 PostgreSQL inference_events
          ↓
웹 /edge-device

- 모델·GPIO는 Pi, DB·API는 PC에서 실행
- 같은 event_id는 중복 저장하지 않음
- 7일차에 PC 플랫폼 Docker/Compose로 확장
````

---

# 6. 오늘 작성한 내용의 최종 정리 및 GitHub 업로드

## 6.1. 오늘의 최종 결과

5일차에 만든 ONNX 모델을 **다시 학습하지 않고** Pi ARM64 CPU로 실행했습니다. 모델은 `normal`, `warning_red`를 예측하며, Confidence가 낮거나 카메라·추론 오류가 발생하면 `UNKNOWN`으로 처리합니다. Pi의 GPIO LED로 결과를 표시하고, 결과 JSON은 PC의 **기존 FastAPI·PostgreSQL**에 기록합니다.

```text
5일차 TinyCNN ONNX + Manifest
      ↓
Raspberry Pi USB Camera → ONNX Runtime CPU
      ↓
normal / warning_red / UNKNOWN
      ↓
GPIO 파랑 / 빨강 / 모두 OFF
      ↓
Pi 로컬 Outbox JSON → HTTP POST
      ↓
교과 13 PC FastAPI → 2일차 PostgreSQL
      ↓
/edge-device 웹 기록 조회 → 7일차 Docker
```

현재 교육용 빨간 카드 분류와 제조 불량·산업안전 판단은 서로 다른 문제입니다. **이번 실습으로 산업안전 탐지 성능을 입증했다고 해석하지 않습니다.**

## 6.2. 오늘의 자가 체크리스트

- [ ] 5일차 `deploy/day05_raspberrypi/`에 모델·설정·Manifest가 준비되어 있습니다.
- [ ] 6일차 `python -m scripts.16_package_day06`으로 배포 패키지를 생성했습니다.
- [ ] Raspberry Pi에서 ARM64·Python·ONNX Runtime CPU 환경을 확인했습니다.
- [ ] `python verify_package.py`의 SHA256 검증을 통과했습니다.
- [ ] Pi에서 Test 이미지 2장으로 실제 ONNX 추론을 수행했습니다.
- [ ] GPIO17 파란 LED, GPIO27 빨간 LED, UNKNOWN의 모두 OFF를 확인했습니다.
- [ ] Pi에서 PC API 주소에 접속하여 장치 결과를 전송했습니다.
- [ ] 2일차 PostgreSQL 기존 테이블에 장치 이벤트가 저장되었습니다.
- [ ] `/edge-device` 화면에서 Pi 이벤트 JSON을 조회했습니다.
- [ ] PC 서버 중단 및 복구 후 Outbox 재전송을 확인했습니다.
- [ ] 6일차 신규 테스트 10개가 통과했습니다.
- [ ] README·구조도·운영 메모를 작성했습니다.
- [ ] 개인 영상·모델·비밀 키가 Git 업로드 목록에 포함되지 않았습니다.

## 6.3. Git 커밋을 남깁니다

PC 프로젝트 최상위에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git status --short
```

5일차 `.gitignore`에 모델과 배포 폴더를 제외했습니다. 6일차에도 생성된 ONNX·장치별 설정·Outbox를 제외합니다.

**`.gitignore` 맨 아래에 추가**

```gitignore
# 6일차 장치별 배포 파일, 비밀 설정, 미전송 JSON
deploy/day06_raspberrypi/
**/device_config.json
**/outbox/
```

**오늘 새 소스와 문서만** Git에 추가합니다.

```bash
git add .gitignore README.md docs/architecture.md docs/day06_pi_runbook.md \
  app/main.py app/edge_device_api.py app/edge_device_repo.py \
  device/__init__.py device/day06_pi/__init__.py \
  device/day06_pi/gpio_led.py device/day06_pi/outbox.py \
  device/day06_pi/verify_package.py device/day06_pi/run_camera.py \
  device/day06_pi/device_config.example.json device/day06_pi/requirements_pi.txt \
  scripts/16_package_day06.py web/pi.html tests/test_day06.py

git status
```

`git status`에서 `.env`, 실제 ONNX·PT 모델, 이미지·영상·Outbox 파일 등이 Staged에 없는지 확인합니다. **이미 Git에서 추적하던 파일은 `.gitignore`만으로 제외되지 않습니다.**

```bash
git commit -m "교과13 6일차: Raspberry Pi ONNX CPU 추론 및 GPIO·PC DB 연동"
git log -1 --oneline
git status --short
```

원격 Git이 허용된 경우에만 다음 명령으로 반영합니다.

```bash
git remote -v
git push origin main
```

폐쇄망에서는 기관이 승인한 내부 Git 서버 또는 로컬 Git을 사용합니다. **실제 Raspberry Pi 배포 파일은 GitHub가 아니라 승인된 별도 전달 방식**으로 보관해야 합니다.

## 6.4. 7일차 수업과 연결합니다

6일차까지 만든 하나의 `subject13_ai_platform` 프로젝트는 **PC의 FastAPI·PostgreSQL·Redis/Celery**와 **Raspberry Pi의 ONNX 추론·GPIO**로 역할이 분리되어 있습니다.

**다음 7일차에서는 PC의 FastAPI·PostgreSQL·Redis·Celery부터 Docker/Compose 실행환경으로 옮깁니다.** Pi GPIO 코드를 갑자기 Docker 컨테이너로 옮기지 않습니다. PC의 API 주소가 바뀌었다면 Pi의 `device_config.json`의 `api_base`만 새 주소로 수정하여 연결을 유지합니다.

**6일차 완료:** 5일차 모델을 Raspberry Pi에서 실제 실행하고, 하드웨어 LED와 기존 PostgreSQL 기록까지 하나의 플랫폼으로 연결하는 방법을 실습했습니다.

