## 1. 오늘의 목표

9일차에는 Jetson에서 TensorRT로 실제 Pose 모델을 실행했고, 10일차에는 그 서비스의 FPS·추론시간·오류와 PC FastAPI 요청을 Prometheus·Grafana에서 관찰했습니다. 오늘은 **같은 교과 13 플랫폼을 AWS EC2 GPU 서버에서도 실행**해 봅니다.

수업을 마치면 다음 작업을 직접 수행할 수 있습니다.

- AWS 계정에서 GPU EC2 인스턴스, 키 페어, 보안 그룹과 과금 조건을 확인합니다.
- 7일차의 `compose.yaml`과 10일차의 `compose.monitoring.yaml`을 유지하면서 Cloud GPU 서비스를 추가합니다.
- 교과 10에서 사용한 **실제 `yolo11n-pose.pt`**를 EC2로 복사하고 모델 파일의 SHA256을 대조합니다.
- EC2 CUDA GPU에서 입력 JPG/PNG를 추론하여 Bounding Box·COCO 17 Keypoint·실제 추론시간을 반환합니다.
- 결과를 1~2일차부터 사용하는 `InferenceEvent` 및 기존 PostgreSQL의 `inference_events` 테이블에 기록합니다.
- 웹 화면에서 Pose와 JSON을 확인하고, Prometheus·Grafana에서 실제 Cloud 지표를 관찰합니다.
- SSH 터널, 비밀키 보관, 백업·복원, 비용 정리와 자동 테스트를 마친 뒤 Git에 변경을 기록합니다.

**오늘 완성할 결과:** 브라우저의 `http://127.0.0.1:18000/cloud`에서 이미지를 선택하면 **EC2 GPU가 새로 추론**하고, `cloud:` 접두사의 실제 결과가 AWS PostgreSQL에 저장됩니다. `http://127.0.0.1:13000`의 Grafana에는 GPU 추론 지표가 수집됩니다. 이 주소는 아래 SSH 터널을 연 **본인 PC 기준**입니다.

**오늘의 Pose 결과는 사람 검출·Keypoint 추정입니다.** 6개 동작의 Random Forest 분류를 실행하는 것은 아니므로 `decision=UNKNOWN`으로 저장합니다. 산업안전 위험/안전을 임의로 OK·NG라고 표시하지 않습니다.

---

## 2. 오늘 수업의 전체 흐름

![[Pasted image 20261018132135.png]]

```text
[10일차 기존 PC 플랫폼]                [교과 10/5일차 모델]
FastAPI / PostgreSQL                 yolo11n-pose.pt
Redis / Celery                              │
Prometheus / Grafana                        │
         │                                  │
         ├──── 기존 DB 백업 ─────┐           │
         │                      │           │
         ▼                      ▼           ▼
   AWS EC2 x86_64 GPU 서버 / Ubuntu DLAMI / Docker
         │
         ├─ 기존 Compose 4개 서비스: api · db · redis · worker
         ├─ 기존 모니터링 2개 서비스: prometheus · grafana
         └─ 신규 GPU 서비스: cloud-gpu (CUDA + YOLO11n-Pose)
                       │
  로컬 PC 브라우저 ── SSH 터널 ── FastAPI /cloud
                       │                  │
                       │            JPG/PNG 업로드
                       │                  ↓
                       │            cloud-gpu 실제 추론
                       │                  ↓
                       ├──── Bounding Box · COCO17 · Confidence
                       │                  ↓
                       └──── 기존 inference_events PostgreSQL 저장
                                          │
                            /metrics → Prometheus → Grafana
                                          │
                            장애·성능·DB 기록 확인
                                          │
                                README · Git · EC2 종료
                                          ↓
                               [12일차] CI/CD · Rollback
```

---

## 3. 실습 전에 알아야 할 핵심

### 3.1. 오늘 그대로 이어받을 것

| 이전 일차 | 그대로 사용하는 결과물 | 오늘 사용할 위치 |
|---|---|---|
| 1일차 | `InferenceEvent` 공통 규격 | `app/schemas.py` |
| 2일차 | ERD·PostgreSQL·SQLAlchemy·CRUD | `app/db.py`, `app/models.py`, `app/repository.py` |
| 3일차 | Redis·Celery · 데이터 내보내기 | `app/celery_app.py`, `data/day03/exports/` |
| 5일차 | 교과 10 기반 YOLO11n-Pose 가중치 | `artifacts/day05/jetson/yolo11n-pose.pt` |
| 7일차 | PC 서비스 Docker Compose | `Dockerfile`, `compose.yaml`, `requirements.docker.txt` |
| 8일차 | Jetson Pose 응답 형식·DB 저장 패턴 | `app/jetson_api.py`, `app/jetson_repo.py` |
| 9일차 | 실제 TensorRT Pose와 카메라 서비스 | `deploy/day08_jetson/` (유지, 클라우드로 복사하지 않음) |
| 10일차 | 운영 지표와 Grafana | `compose.monitoring.yaml`, `monitoring/` |

**이전 결과가 없는 경우:** 해당 일차에 작성한 소스코드와 PostgreSQL 기록 백업, `yolo11n-pose.pt` 원본이 필요합니다. 결석했다면 완료된 교과 13 소스코드, DB 백업과 **교과 10에서 실제 사용한 가중치 파일**을 제공받아 동일한 경로에 놓고 실습합니다. **AI 모델이나 DB 결과를 임의로 생성하여 대체하지 않습니다.** 모델 파일이 없다면 오늘 Cloud GPU 추론 단계는 완료할 수 없습니다.

### 3.2. PC·Jetson·Cloud는 무엇이 다를까요?

- **PC 플랫폼:** FastAPI, DB, Redis, Worker, 모니터링을 통합해서 사용합니다.
- **Jetson Orin Nano Super:** ARM64 기반 GPU 장치이며 9일차에 만든 TensorRT `.engine`을 사용합니다.
- **AWS EC2 GPU:** 선택한 인스턴스가 x86_64이므로 **Jetson ARM64 이미지와 TensorRT 엔진을 그대로 복사하지 않습니다.** 오늘은 실제 `.pt` 가중치를 x86_64 CUDA PyTorch 컨테이너에서 사용합니다.

따라서 동일한 `yolo11n-pose` 모델을 쓰더라도 **GPU 모델·런타임·배포 이미지와 추론시간은 서로 다를 수 있습니다.** 오늘 실행 결과의 `runtime=pytorch_cuda`, `device_id=aws-ec2-gpu`를 확인합니다.

### 3.3. 클라우드 환경에서 반드시 지킬 것

- **GPU EC2는 유료입니다.** 교육기관 또는 개인 계정의 실행 권한과 예산을 확인한 다음 생성합니다. GPU 인스턴스 할당량이 부족하면 인스턴스 생성이 거절될 수 있습니다.
- SSH `22/TCP`는 **본인 접속 IP `/32`**만 허용합니다. `8000`, `3000`, `9090`, `8100`, `5432`, `6379`를 인터넷에 공개하지 않습니다.
- 웹·Grafana는 **SSH 로컬 포트 포워딩**으로 접속합니다. 인증키·DB 비밀번호를 Git이나 화면 캡처에 남기지 않습니다.
- 실습용으로 허가받은 이미지만 사용하고 개인정보·민감한 영상·기업 기밀 모델을 허가 없이 AWS에 업로드하지 않습니다.
- **EC2를 Stop하면 인스턴스 계산 요금은 중단되지만 EBS와 일부 부가 리소스 요금은 남을 수 있습니다.** 수업 후 Stop/Terminate/볼륨/스냅샷/공인 IPv4를 별도로 확인합니다.
- 이전 PC PostgreSQL과 AWS PostgreSQL은 **서로 독립적인 데이터베이스**입니다. 백업 시점의 기록을 복원하지만, 이후 새로 생긴 Cloud 기록이 기존 PC DB에 자동 동기화되는 것은 아닙니다.

### 3.4. 오늘의 배포 범위와 측정 범위

새로 배포한 Cloud 서비스는 **JPG/PNG 단일 이미지에 대한 실제 Pose 추론**을 수행합니다. IP Webcam의 연속 스트림은 9일차 Jetson 실습 결과로 유지하며, 오늘 EC2에 실시간 카메라가 연결되어 있다고 가정하지 않습니다. Jetson이 사설망에 있고 EC2에서 접근할 수 없다면 `platform_jetson_up=0`이어도 Cloud GPU 추론 성공 여부와는 별개입니다.

오늘 수집하는 GPU 추론시간은 **모델 호출 + CUDA 동기화 시간**이고, 브라우저 파일 업로드·HTTP 왕복·DB 저장까지 포함한 전체 체감시간과 다릅니다. GPU 사용률(%)은 새로 수집하지 않으며 미측정값을 그래프에 채우지 않습니다.

---

## 4. 오늘 사용할 기술 스택

| 도구·서비스 | 오늘 하는 일 |
|---|---|
| AWS EC2 `g6.2xlarge` 등 x86_64 GPU 인스턴스 | 학습한 모델을 실제 CUDA GPU에서 실행 |
| AWS Deep Learning Base OSS Nvidia Driver GPU AMI (Ubuntu 24.04) | NVIDIA 드라이버·Docker·GPU 실행환경 준비 |
| SSH · SCP · rsync | 안전한 서버 접속·소스 및 모델 복사 |
| Docker · Docker Compose | 이전 7·10일차 서비스 유지 + GPU 서비스 추가 |
| PyTorch · Ultralytics YOLO11n-Pose · OpenCV | 실제 이미지 Pose 추론·BBox·COCO17 추출 |
| FastAPI · HTTPX · Pydantic | Cloud GPU 결과 검증·전달 |
| SQLAlchemy · PostgreSQL | 기존 `inference_events`에 Cloud 결과 저장 |
| Prometheus · Grafana | Cloud GPU 추론횟수·p95 지연시간·서비스 상태 관찰 |
| Pytest · Git | 이전 코드 회귀 검증 및 일차별 작업 기록 |

AWS 인스턴스 크기·GPU·요금·할당량은 계정과 리전에 따라 다릅니다. **예시 `g6.2xlarge` 사용 가능 여부와 시간당 요금은 인스턴스 생성 화면에서 직접 확인**합니다. AWS AMI는 최신 이미지로 선택하고 정확한 날짜·AMI ID를 고정해서 외우지 않습니다.

---

# 5. 처음부터 끝까지 따라 하는 상세 실습

## PART 1. 기존 프로젝트를 확인하고 11일차 폴더를 준비합니다

### 실습 1. 10일차 프로젝트를 다시 엽니다

오늘도 새 프로젝트를 만들지 않습니다. **기존 PC(WSL Ubuntu)** 터미널에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
pwd
ls app/main.py compose.yaml compose.monitoring.yaml
ls monitoring/prometheus.yml monitoring/grafana/dashboards/
ls -lh artifacts/day05/jetson/yolo11n-pose.pt
code .
```

`subject13_ai_platform` 프로젝트 루트가 맞는지 확인합니다. 마지막 `ls`에서 `.pt` 파일이 없다면 교과 10·5일차의 실제 원본 모델을 준비해야 합니다.

### 실습 2. 기존 PC 플랫폼과 PostgreSQL 기록을 확인합니다

**실습 전 확인:** 10일차 PC의 Compose 서비스와 PostgreSQL 데이터를 확인합니다. 기존 서비스를 삭제하거나 볼륨을 초기화하지 않습니다.

```bash
docker compose -f compose.yaml -f compose.monitoring.yaml ps
curl -s http://127.0.0.1:8000/health
curl -s http://127.0.0.1:8000/api/v1/summary
```

이어서 실제 DB 기록 수를 확인합니다.

```bash
docker compose -f compose.yaml -f compose.monitoring.yaml exec -T db \
  sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT COUNT(*) FROM inference_events;"'
```

`inference_events` 건수는 개인마다 다를 수 있습니다. **현재 PC에 있는 기록 수를 메모**해 두면 나중에 Cloud PostgreSQL로 복원했는지 확인할 수 있습니다. DB가 실행되지 않으면 7일차 교재의 복구 절차부터 수행합니다.

### 실습 3. PostgreSQL 백업 파일을 만듭니다

AWS는 처음에 비어 있는 PostgreSQL을 갖게 됩니다. 2~10일차까지 작성한 데이터베이스를 새 서버에서 그대로 사용하려면 **기존 PC의 실제 DB를 덤프**해야 합니다.

**의사코드**

```text
PC PostgreSQL에 접속
      ↓
inference_events 등 기존 테이블 확인
      ↓
pg_dump -Fc 백업 생성
      ↓
pg_restore -l 로 백업 구조 확인
      ↓
AWS에 전송할 때까지 로컬에서 안전하게 보관
```

```bash
mkdir -p backups/day11

docker compose -f compose.yaml -f compose.monitoring.yaml exec -T db \
  sh -c 'pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc' \
  > backups/day11/pc_before_cloud.dump

ls -lh backups/day11/pc_before_cloud.dump
pg_restore -l backups/day11/pc_before_cloud.dump | head -15
```

`pg_restore`가 PC에 없다면 Ubuntu의 PostgreSQL 클라이언트 패키지를 준비해야 합니다. 파일이 **0바이트가 아니고** `pg_restore -l`에 `TABLE`, `TABLE DATA` 항목이 나타나면 백업 구조를 확인한 것입니다. 이 파일에는 실제 검사·동작 데이터가 들어 있으므로 공개 저장소에 올리지 않습니다.

### 실습 4. 11일차에 새로 만들 폴더와 파일을 확인합니다

기존 `app/main.py`, `compose.yaml`, `compose.monitoring.yaml`은 그대로 유지합니다. 다음 파일만 추가하고 `app/main.py`에 Router 2줄을 연결합니다.

```text
subject13_ai_platform/
├── app/
│   ├── main.py                         [기존 파일: Router 연결 추가]
│   ├── cloud_api.py                    [신규: Cloud 연결·API]
│   └── cloud_repo.py                   [신규: PostgreSQL 저장]
├── cloud_gpu/
│   ├── app.py                          [신규: 실제 CUDA 추론]
│   └── Dockerfile.cloud-gpu           [신규: x86_64 GPU Docker]
├── web/
│   └── cloud.html                     [신규: 이미지·Skeleton 표시]
├── compose.cloud.yaml                 [신규: GPU 서비스 추가]
├── compose.cloud.monitoring.yaml      [신규: Prometheus 설정 교체]
├── monitoring/
│   ├── prometheus.cloud.yml           [신규: Cloud GPU 수집]
│   └── grafana/dashboards/
│       └── subject13_day11_cloud.json [신규: Cloud 대시보드]
├── scripts/
│   ├── 11_dc.sh                        [신규: Compose 실행 공통 명령]
│   └── 11_check_cloud_inputs.py       [신규: 모델·설정 점검]
├── tests/
│   └── test_day11_cloud.py            [신규: Cloud 계약·DB·API 테스트]
├── docs/
│   └── day11_cloud_runbook.md         [신규: 실제 실행·장애 기록]
└── backups/day11/                    [실제 DB 백업, Git 제외]
```

아래 명령으로 폴더를 준비합니다.

```bash
mkdir -p cloud_gpu tests docs scripts backups/day11
mkdir -p monitoring/grafana/dashboards
mkdir -p artifacts/day05/jetson
```

---

## PART 2. Cloud GPU 추론과 공통 이벤트 저장 코드를 작성합니다

### 실습 5. Cloud 전용 결과 저장 함수를 작성합니다

8일차에는 `save_jetson_event()`를 사용했습니다. AWS에서 만들어진 기록을 Jetson 기록과 구별하기 위해 **같은 테이블에 `cloud:` 이벤트로 저장하는 함수**를 추가합니다. 기존 `app/models.py`, `app/db.py`, `app/repository.py`를 변경하지 않습니다.

**의사코드**

```text
Cloud GPU 결과 InferenceEvent 받기
       ↓
task=action, source_mode=live_inference 확인
       ↓
cloud: 이벤트인지 확인
       ↓
같은 event_id가 DB에 이미 있는지 확인
       ├─ 이미 존재 → 이전 결과 반환 (중복 저장 X)
       └─ 없으면 → CameraSource·AIModel 참조 확인
                            ↓
                      inference_events 저장
                            ↓
                      INSERT 결과 반환
```

**파일: `app/cloud_repo.py`**

```python
"""11일차: EC2 GPU Pose 결과를 기존 2일차 inference_events 테이블에 저장합니다."""
from datetime import timezone

from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from app.models import CameraSource, InferenceEventRow
from app.repository import get_or_create_model, row_to_event
from app.schemas import InferenceEvent


def save_cloud_event(db: Session, event: InferenceEvent):
    if event.task != "action" or event.source_mode != "live_inference":
        raise ValueError("EC2 GPU의 실시간 추론(action) 기록이어야 합니다.")
    if not event.event_id.startswith("cloud:"):
        raise ValueError("Cloud 이벤트 ID에 cloud: 접두사가 필요합니다.")
    if event.details.get("device_kind") != "aws_ec2_gpu":
        raise ValueError("AWS GPU 출처 정보가 일치하지 않습니다.")

    lookup = (select(InferenceEventRow)
              .options(selectinload(InferenceEventRow.source),
                       selectinload(InferenceEventRow.model))
              .where(InferenceEventRow.event_id == event.event_id))
    try:
        previous = db.scalar(lookup)
        if previous is not None:
            return row_to_event(previous), False
        source = db.scalar(select(CameraSource).where(CameraSource.source_key == "day11-aws-gpu-pose"))
        if source is None:
            source = CameraSource(source_key="day11-aws-gpu-pose",
                                  name="11일차 AWS EC2 GPU 이미지 Pose", source_type="image_file")
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

`save_cloud_event()`는 **2일차 테이블을 새로 만들지 않습니다.** `CameraSource`와 `AIModel`을 연결한 뒤 기존 `InferenceEventRow`에 저장하며, 같은 이벤트 ID로 요청이 반복되면 새 행을 만들지 않습니다. 따라서 `jetson:`과 `cloud:` 기록을 함께 보관할 수 있습니다.

### 실습 6. Cloud GPU 응답을 공통 JSON으로 바꾸는 API를 작성합니다

8일차의 `PoseResponse` 데이터 구조를 재사용합니다. Cloud GPU에서는 `runtime=pytorch_cuda`, `device_id=aws-ec2-gpu`를 반드시 확인하고, 동작분류가 없는 Pose 결과를 `UNKNOWN`으로 기록합니다.

**의사코드**

```text
브라우저에서 JPG/PNG를 FastAPI로 업로드
             ↓
입력 파일 5MB·이미지 형식 확인
             ↓
내부 Docker 네트워크의 cloud-gpu:8100으로 전달
             ↓
YOLO11n-Pose 실제 CUDA 응답 받아 Pydantic 검증
             ↓
pose_to_cloud_event() → 기존 InferenceEvent
             ↓
save_cloud_event() → PostgreSQL
             ↓
입력 이미지의 Skeleton 정보와 저장 이벤트 반환
```

**파일: `app/cloud_api.py`**

```python
"""11일차: Cloud GPU 전용 서비스를 내부망으로 호출해 Pose 결과를 DB에 보관합니다."""
import io
import os
from pathlib import Path

import httpx
from fastapi import APIRouter, Depends, File, HTTPException, Query, UploadFile
from fastapi.responses import FileResponse
from PIL import Image, UnidentifiedImageError
from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from app.db import get_db
from app.jetson_api import PoseResponse
from app.cloud_repo import save_cloud_event
from app.models import InferenceEventRow
from app.repository import row_to_event
from app.schemas import InferenceEvent

router = APIRouter(tags=["day11-aws-gpu"])
ROOT = Path(__file__).resolve().parents[1]
MAX_IMAGE_BYTES = 5 * 1024 * 1024


def cloud_connection() -> tuple[str, str]:
    url = os.getenv("CLOUD_GPU_URL", "").rstrip("/")
    key = os.getenv("CLOUD_GPU_API_KEY", "")
    # 교육용 내부 Docker 서비스 이름만 허용: 사용자에게 URL을 받지 않습니다.
    if url != "http://cloud-gpu:8100" or len(key) < 24 or "이곳에_" in key:
        raise HTTPException(503, "CLOUD_GPU_URL 및 CLOUD_GPU_API_KEY를 확인하세요.")
    return url, key


def pose_to_cloud_event(pose: PoseResponse) -> InferenceEvent:
    if pose.model_id != "yolo11n-pose" or pose.runtime != "pytorch_cuda":
        raise ValueError("실제 YOLO11n-Pose CUDA 추론 응답이 아닙니다.")
    if pose.device_id != "aws-ec2-gpu":
        raise ValueError("AWS GPU 장치 ID가 아닙니다.")
    if pose.num_persons != len(pose.persons):
        raise ValueError("검출 인원 수가 일치하지 않습니다.")
    if any(len(item.keypoints) not in (0, 17) for item in pose.persons):
        raise ValueError("COCO17 Keypoint 수가 올바르지 않습니다.")
    conf = max((person.bbox_confidence for person in pose.persons), default=None)
    reason = "POSE_ONLY_NO_ACTION_DECISION" if pose.persons else "NO_PERSON_DETECTED"
    return InferenceEvent(
        event_id=f"cloud:{pose.event_id}", task="action", source_mode="live_inference",
        source_project="subject13_ai_platform", source_path="artifacts/day05/jetson/yolo11n-pose.pt",
        recorded_at=pose.recorded_at,
        input_ref=f"{pose.device_id}/{Path(pose.input_ref).name}",
        processor_id=pose.model_id,
        processor_version=f"day11-aws-cuda-{pose.model_sha256[:12]}",
        predicted_label="person" if pose.persons else None,
        expected_label=None, confidence=conf, decision="UNKNOWN", reasons=[reason],
        details={
            "device_kind": "aws_ec2_gpu", "device_id": pose.device_id,
            "gpu_name": pose.gpu_name, "model_sha256": pose.model_sha256,
            "inference_ms": pose.inference_ms, "runtime": pose.runtime,
            "input_width": pose.width, "input_height": pose.height,
            "num_persons": pose.num_persons,
            "persons": [person.model_dump(mode="json") for person in pose.persons],
            "note": pose.note,
        },
    )


@router.get("/cloud")
def cloud_page():
    return FileResponse(ROOT / "web" / "cloud.html")


@router.get("/api/v1/cloud/health")
def cloud_health():
    url, _ = cloud_connection()
    try:
        with httpx.Client(timeout=3, trust_env=False) as client:
            response = client.get(url + "/health")
            response.raise_for_status()
            data = response.json()
        return {
            "connected": data.get("status") == "ready" and data.get("gpu_available") is True,
            "gpu_name": data.get("gpu_name"), "runtime": data.get("runtime"),
        }
    except (httpx.HTTPError, ValueError):
        return {"connected": False, "gpu_name": None, "runtime": None}


@router.post("/api/v1/cloud/analyze")
async def cloud_analyze(image: UploadFile = File(...), db: Session = Depends(get_db)):
    url, key = cloud_connection()
    if image.content_type not in {"image/jpeg", "image/png"}:
        raise HTTPException(415, "JPG/PNG만 지원합니다.")
    raw = await image.read(MAX_IMAGE_BYTES + 1)
    if not raw or len(raw) > MAX_IMAGE_BYTES:
        raise HTTPException(413, "이미지가 없거나 5MB를 초과합니다.")
    try:
        with Image.open(io.BytesIO(raw)) as original:
            if original.format not in {"JPEG", "PNG"}:
                raise HTTPException(422, "실제 JPG/PNG 형식이 아닙니다.")
            width, height = original.size
            if width * height > 4096 * 4096:
                raise HTTPException(413, "이미지 해상도가 너무 큽니다.")
            original.verify()
    except (UnidentifiedImageError, OSError, ValueError):
        raise HTTPException(422, "이미지를 열 수 없습니다.") from None
    filename = Path((image.filename or "pose.jpg").replace("\\", "/")).name[:100]
    try:
        async with httpx.AsyncClient(timeout=65, trust_env=False) as client:
            response = await client.post(
                url + "/api/v1/cloud/pose",
                headers={"X-Cloud-Key": key},
                files={"image": (filename, raw, image.content_type)},
            )
        response.raise_for_status()
        pose = PoseResponse.model_validate(response.json())
        event = pose_to_cloud_event(pose)
    except httpx.RequestError as exc:
        raise HTTPException(503, "AWS GPU 추론 컨테이너와 통신할 수 없습니다.") from exc
    except (httpx.HTTPStatusError, ValueError, KeyError) as exc:
        raise HTTPException(502, "AWS GPU 추론 응답 검증에 실패했습니다.") from exc
    stored, created = save_cloud_event(db, event)
    return {"saved": created, "event": stored.model_dump(mode="json")}


@router.get("/api/v1/cloud/events", response_model=list[InferenceEvent])
def cloud_events(limit: int = Query(default=20, ge=1, le=100), db: Session = Depends(get_db)):
    query = (select(InferenceEventRow)
             .options(selectinload(InferenceEventRow.source),
                      selectinload(InferenceEventRow.model))
             .where(InferenceEventRow.event_id.like("cloud:%"))
             .order_by(InferenceEventRow.id.desc()).limit(limit))
    return [row_to_event(item) for item in db.scalars(query).all()]
```

**코드 이해하기**

`CLOUD_GPU_URL`은 교육용 Docker 내부 주소 `http://cloud-gpu:8100`으로 고정하고, 사용자 입력으로 추론 서버 주소를 바꾸지 않습니다. 내부 서비스 요청에는 24자 이상 `CLOUD_GPU_API_KEY`를 `X-Cloud-Key`로 전달합니다. 화면으로는 키를 보내지 않습니다.

`PoseResponse`는 8일차에서 작성한 데이터 검증 클래스를 재사용합니다. 실제 Cloud 추론 결과를 검증할 때 `device_id`, `model_id`, `runtime`, `model_sha256`, `num_persons`, Keypoint 개수를 함께 확인합니다. 10일차까지의 JSON 계약을 유지하므로 새 DB 테이블이 필요하지 않습니다.

**참고:** `POST /api/v1/cloud/analyze`는 이미지가 업로드되면 *즉시 새로운 추론*을 실행합니다. 1일차의 저장 결과 조회(`recorded_result`)와 다릅니다.

### 실습 7. 기존 FastAPI 서버에 Cloud Router를 연결합니다

기존 `app/main.py`를 열고 **1~10일차 import와 include_router를 삭제하지 않습니다.** 상단 import 목록에 다음 한 줄을 추가합니다.

```python
from app.cloud_api import router as cloud_router
```

기존 `app = FastAPI(...)`가 정의된 아래쪽, 다른 `app.include_router(...)`와 같은 위치에 다음을 **한 번만** 추가합니다.

```python
app.include_router(cloud_router)
```

다음으로 Python 문법을 확인합니다.

```bash
python -m compileall -q app/cloud_api.py app/cloud_repo.py app/main.py
```

확인할 새 주소는 `/cloud`, `/api/v1/cloud/health`, `/api/v1/cloud/analyze`, `/api/v1/cloud/events`입니다. **아직 Cloud GPU 컨테이너를 시작하지 않았으므로**, 새 API의 실제 추론은 뒤에서 실행합니다.

### 실습 8. AWS에서 직접 YOLO11n-Pose를 실행하는 파일을 작성합니다

AWS GPU 컨테이너는 기존 모델 가중치를 직접 로드합니다. 모델 파일이 없거나 `torch.cuda.is_available()`이 `False`이면 **CPU 추론으로 조용히 바꾸지 않고 실패**하도록 처리합니다. GPU 컨테이너에서 6개 Action 분류를 학습하거나 실행하지 않습니다.

**의사코드**

```text
컨테이너 시작
       ↓
/models/yolo11n-pose.pt 실제 파일 존재 확인
       ↓
CUDA GPU 사용 가능 여부 확인
       ↓
모델 파일 SHA256 기록 · YOLO Pose 로드
       ↓
실제 Pose Warm-up 검사
       ↓
FastAPI /health 준비 완료
       ↓
이미지 업로드 시 OpenCV 디코딩
       ↓
실제 GPU 모델.predict(device=0)
       ↓
CUDA 동기화 후 추론시간 실측
       ↓
BBox·17 Keypoint·Confidence·장치 정보 JSON 반환
```

**파일: `cloud_gpu/app.py`**

```python
"""11일차: AWS EC2 CUDA GPU에서 YOLO11n-Pose를 실제로 실행하는 컨테이너 서비스."""
import asyncio
import hashlib
import os
import secrets
import threading
from contextlib import asynccontextmanager
from datetime import datetime, timezone
from pathlib import Path
from time import perf_counter
from uuid import uuid4

import cv2
import numpy as np
import torch
from fastapi import FastAPI, File, Header, HTTPException, UploadFile
from prometheus_client import CONTENT_TYPE_LATEST, Counter, Histogram, generate_latest
from fastapi.responses import Response
from ultralytics import YOLO

WEIGHTS = Path(os.getenv("CLOUD_GPU_WEIGHTS", "/models/yolo11n-pose.pt"))
MAX_IMAGE_BYTES = 5 * 1024 * 1024
MODEL = None
MODEL_LOCK = threading.Lock()
GPU_NAME = ""
MODEL_SHA = ""
REQUESTS = Counter("cloud_gpu_pose_requests_total", "Cloud Pose requests", ("result",))
DURATION = Histogram("cloud_gpu_pose_inference_seconds", "Measured model prediction latency",
                     buckets=(.005, .01, .02, .05, .1, .2, .5, 1, 2, 5, 10))


def sha256_file(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as stream:
        for chunk in iter(lambda: stream.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def check_key(value: str | None) -> None:
    expected = os.getenv("CLOUD_GPU_API_KEY", "")
    if len(expected) < 24 or "이곳에_" in expected:
        raise HTTPException(503, "Cloud GPU 인증 키 설정이 없습니다.")
    if not value or not secrets.compare_digest(value, expected):
        raise HTTPException(401, "Cloud GPU 인증에 실패했습니다.")


def load_model() -> None:
    global MODEL, GPU_NAME, MODEL_SHA
    if not WEIGHTS.is_file():
        raise RuntimeError(f"실제 YOLO Pose 가중치 파일이 없습니다: {WEIGHTS}")
    if not torch.cuda.is_available():
        raise RuntimeError("GPU CUDA가 불가능합니다. CPU 대체 추론은 사용하지 않습니다.")
    candidate = YOLO(str(WEIGHTS))
    with torch.inference_mode():
        result = candidate.predict(np.zeros((640, 640, 3), dtype=np.uint8),
                                   device=0, imgsz=640, conf=.25, max_det=5,
                                   verbose=False)[0]
    if result.keypoints is None:
        raise RuntimeError("실제 COCO17 Pose Keypoint 모델이 아닙니다.")
    GPU_NAME = torch.cuda.get_device_name(0)
    MODEL_SHA = sha256_file(WEIGHTS)
    MODEL = candidate


def infer_frame(frame: np.ndarray, input_ref: str) -> dict:
    if MODEL is None:
        raise RuntimeError("Cloud GPU 모델이 로드되지 않았습니다.")
    with MODEL_LOCK, torch.inference_mode():
        begin = perf_counter()
        output = MODEL.predict(frame, device=0, imgsz=640, conf=.25,
                               max_det=5, verbose=False)[0]
        torch.cuda.synchronize()
        elapsed = perf_counter() - begin
    DURATION.observe(elapsed)
    persons = []
    boxes = output.boxes
    if boxes is not None and len(boxes):
        coordinates = boxes.xyxy.cpu().tolist()
        scores = boxes.conf.cpu().tolist()
        coords = output.keypoints.xy.cpu().tolist() if output.keypoints is not None else []
        confidences = (output.keypoints.conf.cpu().tolist()
                       if output.keypoints is not None and output.keypoints.conf is not None else [])
        for idx, (rect, score) in enumerate(zip(coordinates, scores)):
            keypoints = []
            if idx < len(coords):
                for j, xy in enumerate(coords[idx]):
                    cf = confidences[idx][j] if idx < len(confidences) else None
                    keypoints.append([round(float(xy[0]), 2), round(float(xy[1]), 2),
                                      round(float(cf), 4) if cf is not None else None])
            persons.append({"bbox_xyxy": [round(float(x), 2) for x in rect],
                            "bbox_confidence": round(float(score), 4),
                            "keypoints": keypoints})
    return {
        "event_id": str(uuid4()), "recorded_at": datetime.now(timezone.utc).isoformat(),
        "input_ref": Path(input_ref).name, "width": int(frame.shape[1]),
        "height": int(frame.shape[0]), "num_persons": len(persons), "persons": persons,
        "inference_ms": round(elapsed * 1000, 2), "device_id": "aws-ec2-gpu",
        "model_id": "yolo11n-pose", "model_sha256": MODEL_SHA,
        "runtime": "pytorch_cuda", "gpu_name": GPU_NAME,
        "note": "AWS GPU Pose detection only; action/risk classification is not performed",
    }


@asynccontextmanager
async def lifespan(app: FastAPI):
    load_model()   # 시작할 때 실제 모델/GPU 검증에 실패하면 서비스도 실패합니다.
    yield


app = FastAPI(title="Subject 13 AWS CUDA Pose", version="day11", lifespan=lifespan)


@app.get("/health")
def health():
    return {"status": "ready" if MODEL is not None else "not_ready",
            "gpu_available": torch.cuda.is_available(), "gpu_name": GPU_NAME,
            "model_id": "yolo11n-pose", "runtime": "pytorch_cuda",
            "device": "aws_ec2_gpu"}


@app.post("/api/v1/cloud/pose")
async def pose_image(image: UploadFile = File(...),
                     x_cloud_key: str | None = Header(default=None, alias="X-Cloud-Key")):
    check_key(x_cloud_key)
    if image.content_type not in {"image/jpeg", "image/png"}:
        raise HTTPException(415, "JPG/PNG 형식만 지원합니다.")
    raw = await image.read(MAX_IMAGE_BYTES + 1)
    if not raw or len(raw) > MAX_IMAGE_BYTES:
        raise HTTPException(413, "빈 이미지 또는 5MB 초과 이미지입니다.")
    frame = cv2.imdecode(np.frombuffer(raw, dtype=np.uint8), cv2.IMREAD_COLOR)
    if frame is None:
        raise HTTPException(422, "이미지를 해독하지 못했습니다.")
    if frame.shape[0] * frame.shape[1] > 4096 * 4096:
        raise HTTPException(413, "지원하는 해상도를 초과했습니다.")
    try:
        value = await asyncio.to_thread(
            infer_frame, frame, Path((image.filename or "pose.jpg").replace("\\", "/")).name
        )
        REQUESTS.labels("ok").inc()
        return value
    except Exception:
        REQUESTS.labels("error").inc()
        raise HTTPException(500, "GPU 모델 추론에 실패했습니다.") from None


@app.get("/metrics", include_in_schema=False)
def metrics():
    return Response(content=generate_latest(), media_type=CONTENT_TYPE_LATEST)
```

**코드 이해하기**

`load_model()`은 서비스 시작 시 실제 `.pt` 파일을 읽고 YOLO Pose 출력이 준비되는지 검사합니다. `torch.cuda.synchronize()`는 비동기 CUDA 연산이 끝난 시점까지 측정해 `inference_ms`를 계산합니다. 이 시간은 HTTP 업로드·DB 저장을 제외한 **모델 호출시간**입니다.

`/health`는 CUDA GPU 실행 준비 상태를 확인하며, `/metrics`는 Cloud 추론 요청 수와 추론시간 히스토그램을 수집합니다. `/api/v1/cloud/pose`는 비공개 네트워크 안에서만 사용합니다. `X-Cloud-Key`를 모르는 요청은 거절합니다.

출력의 `predicted_label=person`, `decision=UNKNOWN`은 **사람을 검출했다는 정보와 동작을 아직 판정하지 않았다는 정보**를 구분한 것입니다.

### 실습 9. AWS x86_64용 GPU Dockerfile을 작성합니다

Jetson의 `Dockerfile.jetson`은 ARM64 플랫폼과 JetPack에 맞춘 이미지입니다. 오늘은 **EC2 x86_64용 PyTorch CUDA 이미지**를 새로 사용합니다.

**파일: `cloud_gpu/Dockerfile.cloud-gpu`**

```dockerfile
# 11일차: AWS x86_64 GPU 전용. Jetson ARM64 이미지를 재사용하지 않습니다.
FROM pytorch/pytorch:2.5.1-cuda12.4-cudnn9-runtime
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1
WORKDIR /service
RUN apt-get update && apt-get install -y --no-install-recommends libgl1 libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*
RUN python -m pip install --no-cache-dir \
    "ultralytics>=8.3,<9" "fastapi>=0.115,<1" \
    "uvicorn[standard]>=0.30,<1" "python-multipart>=0.0.9,<1" \
    "prometheus-client>=0.20,<1" "opencv-python-headless>=4.8,<5"
COPY cloud_gpu/app.py /service/app.py
EXPOSE 8100
CMD ["python", "-m", "uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8100", "--workers", "1"]
```

`FROM pytorch/pytorch:2.5.1-cuda12.4-cudnn9-runtime`은 GPU용 PyTorch 런타임의 특정 태그를 사용합니다. 이 이미지의 **CUDA 버전과 EC2 호스트 NVIDIA 드라이버가 호환되어야 하므로**, 실제 배포 전 `nvidia-smi`와 GPU 컨테이너 실행으로 확인합니다. 태그가 더는 배포되지 않거나 패키지 충돌이 있다면 본인 환경의 오류를 확인한 후 공식 제공 태그·호환 버전으로 수정합니다.

이 Dockerfile에는 `.pt` 모델을 `COPY`하지 않습니다. 추론에 사용할 가중치는 Compose에서 **읽기 전용 볼륨**으로 연결합니다.

### 실습 10. 기존 Compose에 Cloud GPU 컨테이너를 추가합니다

7일차 `compose.yaml`은 그대로 유지합니다. 10일차 `compose.monitoring.yaml`도 유지하며 **새 `compose.cloud.yaml`을 추가**합니다.

**의사코드**

```text
기존 Compose: db + redis + api + worker
              ↓
10일차 오버레이: prometheus + grafana
              ↓
11일차 오버레이: cloud-gpu 추가
              ↓
api → cloud-gpu:8100 내부 요청
              ↓
모델 파일만 /models/yolo11n-pose.pt로 읽기 전용 마운트
              ↓
cloud-gpu는 외부 포트 공개하지 않음
```

**파일: `compose.cloud.yaml`**

```yaml
# 사용: docker compose -f compose.yaml -f compose.monitoring.yaml -f compose.cloud.yaml ...
# 원래 Compose의 서비스와 데이터를 유지하면서 GPU 컨테이너만 추가합니다.
services:
  api:
    environment:
      CLOUD_GPU_URL: http://cloud-gpu:8100
      CLOUD_GPU_API_KEY: ${CLOUD_GPU_API_KEY:? .env에 CLOUD_GPU_API_KEY를 설정하세요}
    depends_on:
      cloud-gpu: {condition: service_healthy}

  cloud-gpu:
    build:
      context: .
      dockerfile: cloud_gpu/Dockerfile.cloud-gpu
    image: subject13-cloud-gpu:day11
    restart: unless-stopped
    environment:
      CLOUD_GPU_API_KEY: ${CLOUD_GPU_API_KEY:? .env에 CLOUD_GPU_API_KEY를 설정하세요}
      CLOUD_GPU_WEIGHTS: /models/yolo11n-pose.pt
      NVIDIA_VISIBLE_DEVICES: all
      NVIDIA_DRIVER_CAPABILITIES: compute,utility
    volumes:
      - ./artifacts/day05/jetson/yolo11n-pose.pt:/models/yolo11n-pose.pt:ro
    gpus: all
    expose: ["8100"]
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8100/health', timeout=4)"]
      interval: 20s
      timeout: 6s
      retries: 12
      start_period: 120s
    networks: [platform]
```

**확인할 점**

- `gpus: all`은 **AWS의 GPU 컨테이너에만** 설정합니다. PostgreSQL·Redis·Worker에는 GPU가 필요하지 않습니다.
- `api`는 GPU 컨테이너가 실제로 준비될 때까지 기다립니다. 모델 파일이 없거나 GPU 접근에 실패하면 전체 통합 배포도 완료되지 않으므로 해당 오류를 먼저 고칩니다.
- `cloud-gpu`에는 `ports`가 없습니다. 브라우저가 직접 GPU 추론 서버로 접속하지 않고 기존 FastAPI를 경유합니다.
- 기존 `pg_data`, `redis_data`, Grafana 데이터 볼륨을 지우거나 바꾸지 않습니다.

### 실습 11. FastAPI 모니터링에 Cloud GPU 지표를 추가합니다

10일차 `/metrics`를 유지합니다. Prometheus에는 API와 Cloud GPU의 **서로 다른 측정값**을 수집하게 합니다. 기존 10일차 `monitoring/prometheus.yml`은 원본으로 남깁니다.

**파일: `monitoring/prometheus.cloud.yml`**

```yaml
# 10일차 Prometheus 설정을 복제한 뒤 클라우드 GPU 타깃을 추가한 파일입니다.
global:
  scrape_interval: 10s
  evaluation_interval: 10s
rule_files:
  - /etc/prometheus/alert_rules.yml
scrape_configs:
  - job_name: platform_api
    metrics_path: /metrics
    static_configs:
      - targets: ['api:8000']
  - job_name: cloud_gpu
    metrics_path: /metrics
    static_configs:
      - targets: ['cloud-gpu:8100']
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
```

다음 파일로 10일차 Prometheus 설정만 Cloud 버전으로 바꿉니다. **기존 `compose.monitoring.yaml`은 수정하지 않습니다.**

**파일: `compose.cloud.monitoring.yaml`**

```yaml
# 11일차: 프로메테우스 설정만 Cloud GPU 대상 추가 버전으로 교체
services:
  prometheus:
    volumes:
      - ./monitoring/prometheus.cloud.yml:/etc/prometheus/prometheus.yml:ro
```

**운영 지표의 의미**

- `up{job="cloud_gpu"}`: Prometheus가 `/metrics`에 접근할 수 있는지 여부(0/1). **모델 추론이 성공했다는 뜻은 아닙니다.**
- `cloud_gpu_pose_requests_total{result="ok"}`: 실제 성공한 Pose 요청의 누적 횟수입니다.
- `cloud_gpu_pose_inference_seconds_bucket`: 실측 추론시간의 Prometheus 히스토그램입니다.
- `platform_http_requests_total{route="/api/v1/cloud/analyze"}`: FastAPI에 들어온 HTTP 요청 수입니다. 모델 추론시간과는 다른 지표입니다.

`platform_jetson_up`은 Jetson이 EC2에서 접근 가능하지 않으면 `0`으로 남을 수 있습니다. Cloud API가 정상인지를 확인할 때는 위 Cloud 지표를 별도로 보세요.

### 실습 12. Grafana Cloud GPU 대시보드를 작성합니다

10일차 Grafana가 `monitoring/grafana/dashboards/`의 JSON을 자동 불러오도록 구성되어 있으므로 **같은 폴더에 Cloud 전용 대시보드 한 개만 추가**합니다.

**파일: `monitoring/grafana/dashboards/subject13_day11_cloud.json`**

```json
{
  "uid": "subject13-day11-cloud",
  "title": "교과13 · 11일차 AWS GPU 모니터링",
  "schemaVersion": 40,
  "version": 1,
  "refresh": "10s",
  "timezone": "browser",
  "time": {
    "from": "now-30m",
    "to": "now"
  },
  "panels": [
    {
      "id": 1,
      "type": "timeseries",
      "title": "Cloud GPU 동작 상태 (1=UP)",
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "refId": "A",
          "expr": "up{job=\"cloud_gpu\"}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "short"
        }
      },
      "gridPos": {
        "x": 0,
        "y": 0,
        "w": 12,
        "h": 8
      }
    },
    {
      "id": 2,
      "type": "timeseries",
      "title": "Cloud GPU 분석 요청 (건/초)",
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "refId": "A",
          "expr": "sum(rate(cloud_gpu_pose_requests_total{result=\"ok\"}[5m]))"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps"
        }
      },
      "gridPos": {
        "x": 12,
        "y": 0,
        "w": 12,
        "h": 8
      }
    },
    {
      "id": 3,
      "type": "timeseries",
      "title": "Cloud GPU Pose 추론 p95 (ms)",
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "refId": "A",
          "expr": "1000 * histogram_quantile(0.95, sum by (le) (rate(cloud_gpu_pose_inference_seconds_bucket[5m])))"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "ms"
        }
      },
      "gridPos": {
        "x": 0,
        "y": 8,
        "w": 12,
        "h": 8
      }
    },
    {
      "id": 4,
      "type": "timeseries",
      "title": "Cloud 분석 API HTTP 요청 (건/초)",
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "refId": "A",
          "expr": "sum(rate(platform_http_requests_total{route=\"/api/v1/cloud/analyze\"}[5m]))"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps"
        }
      },
      "gridPos": {
        "x": 12,
        "y": 8,
        "w": 12,
        "h": 8
      }
    }
  ],
  "tags": [
    "subject13",
    "cloud",
    "gpu"
  ]
}
```

대시보드의 네 그래프는 Cloud GPU 수집 상태, 분석 성공 건수, GPU 모델 추론 p95, FastAPI Cloud 분석 HTTP 요청률을 보여줍니다. 아직 이미지를 분석하지 않았다면 추론 그래프가 비어 있을 수 있습니다. 값을 임의로 넣지 않고 실제로 이미지를 분석한 뒤 다시 확인합니다.

### 실습 13. Cloud GPU 웹 화면을 작성합니다

이전 8~9일차에서 사용한 BBox·Skeleton 표시 방식을 유지하면서 Cloud 기록을 한 페이지에서 확인할 수 있게 합니다. 브라우저가 선택한 이미지는 `/api/v1/cloud/analyze`로 전송하고, 응답의 Keypoint로 Canvas에 선을 그립니다.

**의사코드**

```text
/cloud 페이지 열기
        ↓
GPU 연결 상태 /api/v1/cloud/health 요청
        ↓
JPG/PNG 파일 선택 → FastAPI에 POST
        ↓
AWS GPU Pose 결과·DB 저장 응답 수신
        ↓
입력 이미지의 Bounding Box·17 Keypoint 렌더링
        ↓
결과 JSON과 Cloud 기록 목록 표시
```

**파일: `web/cloud.html`**

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>교과 13 · 11일차 AWS GPU Pose</title>
  <style>
    body{font-family:system-ui,sans-serif;max-width:960px;margin:30px auto;padding:0 15px;color:#243448}
    section{border:1px solid #d5dbe2;border-radius:10px;padding:15px;margin:16px 0}
    button,input{font:inherit;padding:8px;margin:5px}
    canvas{display:block;width:100%;height:auto;max-width:850px;border:1px solid #d1d8e0}
    pre{background:#f4f6f9;white-space:pre-wrap;overflow-wrap:anywhere;padding:12px}
    .notice{color:#735f19}
  </style>
</head>
<body>
  <h1>11일차 · AWS EC2 GPU Pose</h1>
  <p class="notice">실제 YOLO11n-Pose GPU 이미지 분석입니다. 사람 검출과 Keypoint만 수행하며 6개 Action·위험행동 판정은 하지 않습니다.</p>
  <section>
    <p id="state">GPU 서버 상태 확인 중…</p>
    <label for="image">직접 촬영했거나 사용 허가를 받은 JPG/PNG</label>
    <input id="image" type="file" accept="image/jpeg,image/png">
    <button id="run">AWS GPU로 분석하고 DB 저장</button>
    <p id="message" role="status"></p>
  </section>
  <section>
    <h2>분석한 이미지 (Bounding Box / COCO17 Skeleton)</h2>
    <canvas id="canvas" width="640" height="360"></canvas>
    <pre id="result">이미지를 선택하고 분석 버튼을 누르세요.</pre>
  </section>
  <section>
    <h2>PostgreSQL에 저장된 Cloud 기록</h2>
    <button id="refresh">목록 새로고침</button>
    <pre id="history">아직 조회하지 않았습니다.</pre>
  </section>
  <script>
    const $=id=>document.getElementById(id);
    const COCO_EDGES=[[5,7],[7,9],[6,8],[8,10],[5,6],[5,11],[6,12],
      [11,12],[11,13],[13,15],[12,14],[14,16],[0,1],[0,2],[1,3],[2,4]];
    async function refreshStatus(){
      try{
        const r=await fetch('/api/v1/cloud/health');
        const d=await r.json();
        $('state').textContent=d.connected ? `GPU 연결됨: ${d.gpu_name} / ${d.runtime}` : 'GPU 연결 안 됨 — 컨테이너 로그를 확인하세요.';
      }catch(e){$('state').textContent='GPU 상태 조회 오류: '+e.message;}
    }
    async function refreshHistory(){
      try{
        const r=await fetch('/api/v1/cloud/events?limit=15');
        if(!r.ok)throw Error('HTTP '+r.status);
        const rows=await r.json();
        $('history').textContent=rows.length?rows.map(x=>
          `${x.recorded_at} | ${x.input_ref} | ${x.decision} | ${x.details?.gpu_name||''}`
        ).join('\n'):'아직 저장된 Cloud 추론 결과가 없습니다.';
      }catch(e){$('history').textContent='DB 조회 실패: '+e.message;}
    }
    function drawPose(img,pose){
      const canvas=$('canvas'); canvas.width=img.naturalWidth; canvas.height=img.naturalHeight;
      const ctx=canvas.getContext('2d');ctx.drawImage(img,0,0);
      for(const p of pose.persons){
        const b=p.bbox_xyxy;
        ctx.strokeStyle='#ff4d49';ctx.lineWidth=3;
        ctx.strokeRect(b[0],b[1],b[2]-b[0],b[3]-b[1]);
        const points=p.keypoints||[];
        ctx.strokeStyle='#13dc7a';ctx.lineWidth=3;
        for(const [a,z] of COCO_EDGES){
          const v=points[a],w=points[z];
          if(!v||!w||(v[2]??0)<0.25||(w[2]??0)<0.25)continue;
          ctx.beginPath();ctx.moveTo(v[0],v[1]);ctx.lineTo(w[0],w[1]);ctx.stroke();
        }
        ctx.fillStyle='#ffe14d';
        for(const v of points){
          if(!v||(v[2]??0)<0.25)continue;
          ctx.beginPath();ctx.arc(v[0],v[1],3,0,Math.PI*2);ctx.fill();
        }
      }
    }
    function loadImage(file){
      return new Promise((resolve,reject)=>{
        const url=URL.createObjectURL(file);const img=new Image();
        img.onload=()=>{URL.revokeObjectURL(url);resolve(img);};
        img.onerror=()=>{URL.revokeObjectURL(url);reject(Error('이미지 표시 불가'));};
        img.src=url;
      });
    }
    $('run').addEventListener('click',async()=>{
      const file=$('image').files[0];
      if(!file){$('message').textContent='JPG/PNG 파일을 선택하세요.';return;}
      if(!['image/jpeg','image/png'].includes(file.type)||file.size>5*1024*1024){
        $('message').textContent='5MB 이하 JPG/PNG만 가능합니다.';return;
      }
      $('run').disabled=true;$('message').textContent='GPU 추론 중…';
      try{
        const data=new FormData();data.append('image',file);
        const response=await fetch('/api/v1/cloud/analyze',{method:'POST',body:data});
        if(!response.ok)throw Error(`HTTP ${response.status}: ${await response.text()}`);
        const result=await response.json();
        const img=await loadImage(file);
        drawPose(img,result.event.details);
        $('result').textContent=JSON.stringify(result.event,null,2);
        $('message').textContent=`DB ${result.saved?'신규 저장':'중복 기록'} · 사람 ${result.event.details.num_persons}명 · ${result.event.details.inference_ms}ms`;
        await refreshHistory();
      }catch(e){$('message').textContent='분석 실패: '+e.message;}
      finally{$('run').disabled=false;}
    });
    $('refresh').addEventListener('click',refreshHistory);
    refreshStatus();refreshHistory();
  </script>
</body>
</html>
```

**코드 이해하기**

`fetch()`로 같은 주소의 FastAPI에 요청하므로 브라우저에 Cloud GPU 내부 주소나 비밀키를 공개하지 않습니다. `FormData`를 사용해 파일을 업로드하고 응답 JSON의 `details.persons`에 담긴 Bounding Box와 17개 Keypoint를 그립니다. 사진에서 실제 사람이 검출되지 않으면 `num_persons=0`과 `UNKNOWN`을 표시합니다.

### 실습 14. Compose 실행 명령을 한 파일로 정리합니다

매번 네 개의 Compose 파일 순서를 기억하기 어렵기 때문에 **항상 같은 순서**로 실행하는 작은 스크립트를 만듭니다.

**파일: `scripts/11_dc.sh`**

```bash
#!/usr/bin/env bash
# 11일차 AWS EC2에서 7·10·11일차 Compose 파일을 항상 같은 순서로 결합합니다.
set -euo pipefail
ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$ROOT"
exec docker compose \
  -f compose.yaml \
  -f compose.monitoring.yaml \
  -f compose.cloud.yaml \
  -f compose.cloud.monitoring.yaml \
  "$@"
```

앞으로 AWS EC2에서는 다음처럼 사용합니다.

```bash
bash scripts/11_dc.sh config --quiet
bash scripts/11_dc.sh ps
bash scripts/11_dc.sh logs --tail=30 api
```

`config --quiet`은 Compose 구성 검증만 수행하므로 **비밀번호가 들어 있는 환경변수 내용이 터미널에 길게 출력되지 않도록** 할 수 있습니다.

### 실습 15. 모델 파일과 Cloud 코드 사전 검사를 작성합니다

GPU 컨테이너를 만들기 전에 파일 누락이 없는지 확인하는 프로그램을 작성합니다. 실제 모델을 임의의 랜덤 가중치로 대체하지 않습니다.

**파일: `scripts/11_check_cloud_inputs.py`**

```python
"""11일차: AWS 배포 전에 누락 파일과 모델의 SHA256을 확인합니다."""
import hashlib
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]
REQUIRED = [
    "compose.yaml", "compose.monitoring.yaml",
    "compose.cloud.yaml", "compose.cloud.monitoring.yaml",
    "cloud_gpu/Dockerfile.cloud-gpu", "cloud_gpu/app.py",
    "app/cloud_api.py", "app/cloud_repo.py", "web/cloud.html",
    "monitoring/prometheus.cloud.yml", ".env",
    "artifacts/day05/jetson/yolo11n-pose.pt",
]


def main() -> None:
    missing = [name for name in REQUIRED if not (ROOT / name).is_file()]
    if missing:
        raise SystemExit("필수 파일 누락:\n" + "\n".join(" - " + x for x in missing))
    file = ROOT / "artifacts/day05/jetson/yolo11n-pose.pt"
    if file.stat().st_size < 100_000:
        raise SystemExit("Pose 가중치 파일 크기가 비정상적입니다. 원본 파일인지 확인하세요.")
    digest = hashlib.sha256()
    with file.open("rb") as stream:
        for chunk in iter(lambda: stream.read(1024 * 1024), b""):
            digest.update(chunk)
    print("필수 코드·설정·모델 파일 확인 완료")
    print("모델:", file.relative_to(ROOT))
    print("SHA256:", digest.hexdigest())
    print("※ 이 검사는 GPU 실행/모델 정확도를 확인하지 않습니다.")


if __name__ == "__main__":
    main()
```

이 프로그램은 필수 파일이 존재하는지 확인하고 **실제 가중치 파일의 SHA256**을 출력합니다. 이것만으로 GPU 추론이 정상이라는 뜻은 아니므로, 뒤의 모델 실행과 실제 이미지 분석도 확인해야 합니다.

---

## PART 3. AWS EC2 GPU 인스턴스를 준비하고 기존 프로젝트를 옮깁니다

### 실습 16. AWS 계정의 GPU 사용 권한과 예산을 확인합니다

오늘은 **실제로 과금되는 클라우드 장치**를 사용합니다. AWS 콘솔에 로그인한 뒤 아래 순서대로 확인합니다.

1. AWS Console에서 **사용할 Region**을 선택합니다. 이후 모든 EC2 화면에서 같은 Region을 사용합니다.
2. **Service Quotas → Amazon EC2 → Running On-Demand G and VT instances** 등 실제 G 계열 사용 한도를 확인합니다. 화면의 표기는 계정·리전에 따라 달라질 수 있습니다.
3. AWS EC2 인스턴스 생성 화면에서 **사용할 GPU 인스턴스의 시간당 요금**, EBS 스토리지 요금, 공인 IPv4 관련 요금을 확인합니다.
4. **AWS Budgets**에서 실습용 예산과 알림을 구성합니다. 예산 알림은 과금 초과를 자동으로 반드시 막는 장치가 아니므로 실습 종료 후 리소스를 직접 정리해야 합니다.
5. 학교·기관 계정이라면 실습 계정의 종료 시점, 데이터 반입 정책, 동시 GPU 사용 가능 대수를 확인합니다.

**권장 실습 예시:** `g6.2xlarge` 등 **x86_64 Linux GPU** 인스턴스를 검토합니다. 7개의 서비스가 함께 실행되므로 매우 작은 메모리의 인스턴스보다 여유가 있는 인스턴스가 유리합니다. 실제 타입은 **할당량·가용 여부·예산**에 따라 승인된 것을 선택합니다. 비용 숫자를 교재의 고정값으로 사용하지 않습니다.

공식 AWS EC2 가속 컴퓨팅 사양: https://docs.aws.amazon.com/ec2/latest/instancetypes/ac.html

### 실습 17. GPU용 Ubuntu AMI로 EC2 인스턴스를 생성합니다

AWS Console → **EC2 → Instances → Launch instances**에서 다음을 선택합니다.

| 설정 항목 | 실습에서 확인할 값 |
|---|---|
| Name | `subject13-day11-gpu` 등 학습용 이름 |
| AMI | **Deep Learning Base OSS Nvidia Driver GPU AMI (Ubuntu 24.04)** 최신 x86_64 이미지 |
| Instance type | 권한·Region·예산에 맞는 x86_64 GPU, 예: `g6.2xlarge` |
| Key pair | 새 교육용 키 페어(.pem) 또는 승인을 받은 기존 키 페어 |
| Network | Public IPv4 접속이 가능한 실습 네트워크. 기관 정책이 다르면 승인된 VPN·SSM 경로 사용 |
| Security group inbound | `SSH TCP/22`를 **My IP/32**로만 허용 |
| 금지할 공개 포트 | `8000/3000/9090/8100/5432/6379` 공개 규칙 만들지 않음 |
| EBS root volume | Docker CUDA/PyTorch 이미지와 DB 크기를 고려하여 **용량·요금 확인** (예: 100GiB 이상) |

**중요:** AMI 이름이 비슷한 **ARM64/G5g용 이미지**가 아니라 선택한 GPU 인스턴스의 CPU 아키텍처와 일치하는 **x86_64 AMI**를 선택합니다. 최신 AMI 날짜·정확한 ID는 리전에 따라 바뀔 수 있습니다.

AWS 공식 AMI 확인: https://docs.aws.amazon.com/dlami/latest/devguide/aws-deep-learning-x86-base-gpu-ami-ubuntu-24-04.html

인스턴스 상태가 `Running`이 되고 상태 검사가 통과할 때까지 기다립니다. 생성 후 EC2 상세 화면의 **Public IPv4 address**를 확인합니다. 이 주소는 개인 환경마다 다르므로 문서에 고정하지 않습니다.

### 실습 18. 기존 PC에서 AWS로 SSH 접속합니다

**이 단계의 명령은 기존 PC의 WSL Ubuntu 터미널에서 실행합니다.** 키 파일은 계정에 연결된 개인 파일이므로 `.git` 저장소 안에 두지 않습니다.

```bash
# 실제 EC2 IPv4 주소와 키 파일 경로를 입력하세요.
export EC2_PUBLIC_IP="여기에_실제_EC2_공인_IP"
export SSH_KEY="$HOME/.ssh/subject13-day11.pem"

chmod 400 "$SSH_KEY"
ssh -i "$SSH_KEY" ubuntu@"$EC2_PUBLIC_IP"
```

Ubuntu 계열의 AWS Deep Learning AMI라면 일반적으로 `ubuntu` 계정으로 로그인합니다. 실제 AMI의 연결 화면에 다른 계정이 안내되면 그 계정을 따릅니다. 최초 SSH 연결 시 호스트 키 지문을 확인하고, 확인되지 않은 서버의 지문을 무조건 신뢰하지 않습니다.

로그인에 성공하면 **EC2 SSH 터미널**에서 실행합니다.

```bash
whoami
uname -m
cat /etc/os-release | head -6
nvidia-smi
docker --version
docker compose version
```

- `uname -m`은 `x86_64`여야 합니다.
- `nvidia-smi`에서 실제 GPU 이름과 Driver Version이 나타나야 합니다.
- `docker compose version`이 실패하면 AMI에 Docker Compose 플러그인이 설치되어 있는지 확인합니다.

**드라이버/Toolkit이 없으면 임의의 드라이버 설치 명령을 먼저 실행하지 않습니다.** 선택한 AWS AMI의 공식 드라이버·NVIDIA Container Toolkit 설치 안내에 맞춰 환경을 보완하고 다시 확인합니다.

### 실습 19. EC2의 Docker GPU 연결을 검증합니다

**EC2 SSH 터미널**에서 실행합니다.

```bash
nvidia-smi
sudo docker run --rm --gpus all \
  nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

컨테이너 안에서도 GPU 모델 이름이 출력되면 Docker가 NVIDIA GPU를 전달한 것입니다. 이미지 태그가 사용 가능한지, 호스트 드라이버가 해당 CUDA 컨테이너와 호환되는지는 **실제 실행 결과로 확인**합니다.

`could not select device driver ... gpu` 등 오류가 나타나면 EC2에서 **NVIDIA Container Toolkit**이 준비되어 있는지 확인합니다. Toolkit이 설치되어 있으나 Docker Runtime 등록만 되지 않은 환경에서는 아래의 공식 방식을 사용할 수 있습니다.

```bash
command -v nvidia-ctk
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
sudo docker run --rm --gpus all \
  nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

`nvidia-ctk` 자체가 없다면 선택한 AMI의 패키지/ToolKit 안내를 확인하고 설치해야 합니다. **컨테이너 GPU 연결이 실패하면 다음 추론 실습으로 넘어가지 않습니다.**

NVIDIA 공식 Docker GPU 테스트: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/sample-workload.html

Ubuntu 사용자가 `docker` 명령에 접근 권한이 없을 때는, 수업용 EC2 계정에 필요한 권한이 승인되어 있는지 확인한 후 아래처럼 설정하고 다시 SSH로 접속할 수 있습니다. `docker` 그룹은 높은 시스템 권한을 가지므로 승인된 계정에서만 사용합니다.

```bash
sudo usermod -aG docker "$USER"
```

### 실습 20. 승인된 소스코드를 EC2로 복사합니다

**EC2 SSH 터미널**에서 먼저 대상 경로를 만듭니다.

```bash
mkdir -p ~/ai_vision/subject13_ai_platform
sudo apt-get update
sudo apt-get install -y rsync
```

이제 **기존 PC의 새로운 WSL 터미널**을 열어 다음을 실행합니다. (EC2 SSH 세션은 닫지 않아도 됩니다.)

```bash
cd ~/ai_vision/subject13_ai_platform
sudo apt-get install -y rsync

# 실제 주소와 SSH 키 경로를 이 새 PC 터미널에도 설정합니다.
export EC2_PUBLIC_IP="여기에_실제_EC2_공인_IP"
export SSH_KEY="$HOME/.ssh/subject13-day11.pem"

rsync -av -e "ssh -i $SSH_KEY" \
  --exclude '.git/' \
  --exclude '.venv/' \
  --exclude '.env' \
  --exclude '.env.*' \
  --exclude '.pytest_cache/' \
  --exclude '__pycache__/' \
  --exclude 'backups/' \
  --exclude 'artifacts/' \
  --exclude 'data/' \
  --exclude 'handoff/records/' \
  --exclude 'handoff/provided/' \
  --exclude '*.pem' --exclude '*.key' \
  ./ ubuntu@"$EC2_PUBLIC_IP":~/ai_vision/subject13_ai_platform/
```

**코드 이해하기:** `rsync`는 기존 교과 13의 소스만 복사합니다. `.env`·인증키·원본 이미지·모델·DB 백업은 한꺼번에 올리지 않습니다. 그래서 **기존 프로젝트는 보존되고**, 허가된 자료를 필요한 만큼 따로 전송할 수 있습니다. `--delete`를 사용하지 않으므로 서버에서 작업한 파일을 자동 삭제하지도 않습니다.

기관 정책상 외부 클라우드로 소스를 옮길 수 없다면 여기서 중단하고 **승인된 실습 리포지토리·전용 네트워크·교육용 계정**을 사용하세요. 무단 업로드로 우회하지 않습니다.

### 실습 21. 실제 모델 가중치와 DB 백업을 별도로 전송합니다

앞의 `rsync`에서는 모델과 DB 백업을 제외했습니다. 이번에는 수업에 사용할 실제 자료 중 **반입 승인이 된 것만** EC2에 전송합니다.

먼저 **PC 터미널**에서 원본 파일이 준비되어 있는지 확인합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
ls -lh artifacts/day05/jetson/yolo11n-pose.pt
ls -lh backups/day11/pc_before_cloud.dump
sha256sum artifacts/day05/jetson/yolo11n-pose.pt
```

**EC2 SSH 터미널**에서 받는 폴더를 미리 만듭니다.

```bash
cd ~/ai_vision/subject13_ai_platform
mkdir -p artifacts/day05/jetson backups/day11
mkdir -p handoff/records data/day03/exports
```

**PC 터미널**에서 다음을 실행합니다.

```bash
scp -i "$SSH_KEY" \
  artifacts/day05/jetson/yolo11n-pose.pt \
  ubuntu@"$EC2_PUBLIC_IP":~/ai_vision/subject13_ai_platform/artifacts/day05/jetson/

scp -i "$SSH_KEY" \
  backups/day11/pc_before_cloud.dump \
  ubuntu@"$EC2_PUBLIC_IP":~/ai_vision/subject13_ai_platform/backups/day11/
```

이어서 **EC2 SSH 터미널**에서 검증합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
sha256sum artifacts/day05/jetson/yolo11n-pose.pt
ls -lh backups/day11/pc_before_cloud.dump
```

PC에서 확인한 SHA256과 EC2의 SHA256이 정확히 일치해야 합니다. `sha256sum`이 다른 경우 손상된 파일로 모델을 실행하지 말고 파일을 다시 확인합니다.

**저장 JSON 복원 자료가 따로 필요한 경우에만** `handoff/records/`의 교과 9·10 실제 JSON을 승인된 방식으로 추가 전송합니다. DB 백업을 복원할 경우 **기존 기록을 JSON으로 한 번 더 중복 이관하지 않습니다.**

### 실습 22. EC2에 Cloud 환경변수 `.env`를 작성합니다

10일차의 PC `.env`를 통째로 클라우드에 복사하지 않습니다. **EC2 SSH 터미널**에서 새 `.env`를 작성하고 클라우드용 키를 별도로 설정합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
openssl rand -hex 24  # 강력한 DB 비밀번호 후보. 출력값은 비밀로 보관합니다.
openssl rand -hex 24  # 장치 인증키 후보
openssl rand -hex 24  # Cloud GPU 서비스 내부 인증키 후보
openssl rand -hex 24  # Grafana 비밀번호 후보
nano .env
```

**파일: `.env` (EC2에서 직접 입력, Git 제외)**

```dotenv
DB_NAME=subject13_platform
DB_USER=subject13_user
DB_PASSWORD=이곳에_생성한_실제_DB_비밀번호
DAY06_DEVICE_API_KEY=이곳에_생성한_실제_장치_인증키
CLOUD_GPU_API_KEY=이곳에_생성한_실제_Cloud_인증키
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=이곳에_생성한_실제_Grafana_비밀번호
PLATFORM_BIND_HOST=127.0.0.1
MONITORING_BIND_HOST=127.0.0.1
DAY08_JETSON_URL=
DAY08_JETSON_API_KEY=
```

4일차의 `DAY04_OCR_URL`, `DAY04_ACTION_URL`은 기존 `compose.yaml`에서 PC 호스트용 주소로 정해져 있습니다. **EC2에 해당 OCR/Action 서비스가 없으면 실제 모델 호출이 실패할 수 있습니다.** 오늘은 별도로 추가한 Cloud GPU Pose를 검증하며, **4일차 OCR/Action·6일차 Pi·9일차 Jetson까지 모두 클라우드로 이동했다고 가정하지 않습니다.**

파일 저장 후 다음을 실행합니다.

```bash
chmod 600 .env
ls -l .env
python3 scripts/11_check_cloud_inputs.py
bash scripts/11_dc.sh config --quiet
bash scripts/11_dc.sh config --services
```

`ls -l`에 `.env` 소유자만 읽고 쓸 수 있는 권한이 보이는지 확인합니다. `config --services`에 `db`, `redis`, `api`, `worker`, `prometheus`, `grafana`, `cloud-gpu`가 나타나면 Compose 구성이 결합된 것입니다.

`.env`에는 실제 키를 넣되 **강의 자료에 비밀키를 적거나 화면을 촬영해 공유하지 않습니다.**

---

## PART 4. 실제 PostgreSQL과 Docker 서비스를 AWS에서 실행합니다

### 실습 23. EC2 PostgreSQL·Redis만 먼저 실행합니다

**EC2 SSH 터미널**에서 다음 명령을 입력합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
bash scripts/11_dc.sh up -d db redis
bash scripts/11_dc.sh ps
bash scripts/11_dc.sh logs --tail=30 db
```

왜 DB부터 시작할까요? 기존 PC DB 백업을 **아직 비어 있는 AWS PostgreSQL**에 복원하기 위해서입니다. Cloud GPU 모델과 FastAPI를 먼저 실행하여 테이블이 생성되는 일을 피합니다.

### 실습 24. 기존 PC 데이터베이스를 AWS DB에 한 번만 복원합니다

**중요:** 이 실습은 **새로 생성한 빈 AWS DB에 한 번만** 수행합니다. 이미 AWS DB에 새 결과를 저장했다면 이 명령을 반복하지 마세요. 데이터를 지우는 `docker compose down -v`는 사용하지 않습니다.

**의사코드**

```text
AWS DB의 테이블 목록 확인
         ↓
아직 기록·테이블이 없는 새 DB인지 확인
         ↓
PC에서 만든 pg_dump -Fc 파일 확인
         ↓
pg_restore 실행
         ↓
PC에서 확인한 inference_events 건수 대조
```

**EC2 SSH 터미널**에서 다음을 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
ls -lh backups/day11/pc_before_cloud.dump

# 현재 AWS DB 테이블 확인 (새 DB라면 일반적으로 테이블이 없습니다.)
bash scripts/11_dc.sh exec -T db sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "\\dt"'

# 빈 DB에서 딱 한 번 복원합니다.
bash scripts/11_dc.sh exec -T db sh -c \
  'pg_restore -U "$POSTGRES_USER" -d "$POSTGRES_DB" --no-owner --no-privileges' \
  < backups/day11/pc_before_cloud.dump
```

이어서 실제 데이터 개수를 확인합니다.

```bash
bash scripts/11_dc.sh exec -T db sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT COUNT(*) FROM inference_events;"'

bash scripts/11_dc.sh exec -T db sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT task, COUNT(*) FROM inference_events GROUP BY task ORDER BY task;"'
```

**확인:** 실습 2에서 기록한 기존 PC의 개수와 AWS의 개수가 같아야 합니다. 수업 시간에는 기존 PC DB를 백업한 순간 이후에 새로 추가된 기록은 포함되지 않습니다.

**백업 파일이 전혀 없는 학생:** 2일차의 `scripts/02_create_tables.py`로 빈 테이블을 만들고 `scripts/03_import_records_to_db.py`로 1일차 JSON을 이관하는 방법을 사용합니다. 다만 이 경우 **2~10일차의 기존 DB 결과까지 모두 복원했다는 의미는 아닙니다.** 이미 기록이 있는 DB에는 스크립트를 중복으로 실행하거나 초기화하지 않습니다.

### 실습 25. EC2에 기존 플랫폼 이미지와 GPU 이미지를 빌드합니다

이제 기존 API 이미지와 Cloud GPU 이미지를 빌드합니다. `cloud-gpu`는 용량이 큰 PyTorch·CUDA 이미지를 가져와야 하므로 **스토리지·네트워크·다운로드 시간**을 확인합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
bash scripts/11_dc.sh build api cloud-gpu
bash scripts/11_dc.sh images
```

`No space left on device` 오류가 발생하면 EBS 용량부터 확인합니다. 운영 중인 볼륨이나 이미지를 함부로 지우지 말고 **승인된 스토리지 확장·불필요한 학습용 파일 점검**을 먼저 진행하세요.

### 실습 26. 기존 6개 서비스와 Cloud GPU 서비스를 함께 시작합니다

**EC2 SSH 터미널**에서 다음을 실행합니다.

```bash
bash scripts/11_dc.sh up -d
bash scripts/11_dc.sh ps
bash scripts/11_dc.sh logs --tail=80 cloud-gpu
bash scripts/11_dc.sh logs --tail=60 api
```

`cloud-gpu` 서비스가 준비되지 않으면 `api`가 기다릴 수 있습니다. 우선 `cloud-gpu` 로그에서 `Model file`·CUDA·PyTorch·Ultralytics 오류가 있는지 확인하세요. **CPU로 임시 대체해서 Cloud GPU가 성공했다고 기록하지 않습니다.**

GPU 컨테이너에서 실제 CUDA 사용 여부를 확인합니다.

```bash
bash scripts/11_dc.sh exec cloud-gpu \
  python -c 'import torch; print("CUDA:", torch.cuda.is_available()); print("GPU:", torch.cuda.get_device_name(0) if torch.cuda.is_available() else "NONE")'
```

`CUDA: True`와 실제 GPU 이름이 나타나야 합니다. `cloud-gpu` 컨테이너의 `/health`는 같은 Docker 네트워크의 API 컨테이너에서만 조회합니다.

```bash
bash scripts/11_dc.sh exec -T api python - <<'PYCODE'
import json
from urllib.request import urlopen
with urlopen('http://cloud-gpu:8100/health', timeout=5) as response:
    print(json.dumps(json.load(response), ensure_ascii=False, indent=2))
PYCODE
```

여기서 `-T`는 표준입력으로 전달한 Python 코드를 컨테이너 안에서 그대로 실행하기 위한 옵션입니다. 장치를 확인하기 위해 Cloud GPU 포트 `8100`을 외부에 공개하지 않습니다.

### 실습 27. 외부에 서비스 포트가 노출되지 않았는지 점검합니다

**EC2 SSH 터미널**에서 실행합니다.

```bash
bash scripts/11_dc.sh ps
sudo ss -ltnp | grep -E ':(8000|3000|9090|8100|5432|6379)\b' || true
```

호스트에서 열리는 `8000`, `3000`, `9090`은 **127.0.0.1에만 바인딩**되어 있어야 합니다. `8100`, `5432`, `6379`는 외부에 공개되지 않아야 합니다. Docker 내부 컨테이너 간 통신은 가능합니다.

EC2 보안 그룹 Inbound 규칙도 다시 확인합니다. `8000`, `3000`, `9090`을 실습 편의 때문에 `0.0.0.0/0`으로 열지 않습니다.

### 실습 28. 로컬 PC에서 SSH 터널로 클라우드 웹을 엽니다

**기존 PC의 새로운 WSL 터미널**에서 실행합니다. EC2 SSH 터미널과 구별하세요.

```bash
export EC2_PUBLIC_IP="여기에_실제_EC2_공인_IP"
export SSH_KEY="$HOME/.ssh/subject13-day11.pem"

ssh -i "$SSH_KEY" -N \
  -L 18000:127.0.0.1:8000 \
  -L 13000:127.0.0.1:3000 \
  -L 19090:127.0.0.1:9090 \
  ubuntu@"$EC2_PUBLIC_IP"
```

터널 터미널은 **열어 둔 상태**로 유지합니다. 이제 PC 브라우저에서 주소를 엽니다.

| 로컬 PC 주소 | 연결되는 EC2 서비스 |
|---|---|
| `http://127.0.0.1:18000/` | Cloud FastAPI 기존 플랫폼 |
| `http://127.0.0.1:18000/cloud` | 새 Cloud GPU 이미지 Pose 화면 |
| `http://127.0.0.1:18000/docs` | Cloud API 자동 문서 |
| `http://127.0.0.1:18000/api/v1/cloud/health` | GPU 추론 서비스 상태 |
| `http://127.0.0.1:13000` | Grafana |
| `http://127.0.0.1:19090` | Prometheus |

연결이 되지 않으면 (1) SSH 터널 실행 여부, (2) EC2 `bash scripts/11_dc.sh ps`, (3) `127.0.0.1` 바인딩 설정을 확인합니다. AWS 퍼블릭 IP를 웹 주소에 직접 쓰는 방식이 아닙니다.

---

## PART 5. 실제 추론·PostgreSQL·Grafana 결과를 검증합니다

### 실습 29. 기존 FastAPI와 Cloud GPU 상태를 먼저 확인합니다

**기존 PC 브라우저**에서 다음 두 주소를 엽니다.

```text
http://127.0.0.1:18000/health
http://127.0.0.1:18000/api/v1/cloud/health
```

`/health`에서는 기존 FastAPI가 동작하는지 확인합니다. `/api/v1/cloud/health`에서 `connected: true`, `runtime: pytorch_cuda` 및 실제 GPU 이름이 나타나는지 확인합니다.

**현재 PC의 실제 교과 9·10·Pi·Jetson 장비 서비스는 EC2와 별개의 환경**입니다. EC2에서 `/health`가 정상이어도 외부 장치가 자동으로 AWS에 연결된 것은 아닙니다.

### 실습 30. Cloud GPU에 실제 이미지를 업로드하고 Pose를 확인합니다

**PC 브라우저**에서 `http://127.0.0.1:18000/cloud`를 엽니다.

1. 본인이 촬영했거나 사용 권한이 있는 **JPG/PNG 사진**을 선택합니다.
2. **AWS GPU로 분석하고 DB 저장** 버튼을 누릅니다.
3. 이미지 위에 Bounding Box와 COCO 17 Keypoint가 표시되는지 확인합니다.
4. JSON에서 `event_id`, `input_ref`, `model_sha256`, `runtime`, `inference_ms`, `persons`를 찾습니다.
5. `event_id`가 `cloud:`로 시작하고 `details.device_kind`가 `aws_ec2_gpu`인지 확인합니다.
6. `decision`이 `UNKNOWN`인지, 이유가 `POSE_ONLY_NO_ACTION_DECISION` 또는 `NO_PERSON_DETECTED`인지 확인합니다.

```text
실제 업로드한 이미지
       ↓
AWS Cloud GPU: YOLO11n-Pose (pytorch_cuda)
       ↓
BBox + COCO17 + 실측 inference_ms
       ↓
InferenceEvent(event_id=cloud:...)
       ↓
PostgreSQL 저장
       ↓
/cloud 화면에 결과 표시
```

**주의:** 사진에 사람이 없어도 추론 자체가 성공했을 수 있으며 `num_persons=0`과 `UNKNOWN`이 나오는 것이 정상입니다. 임의의 신뢰도 숫자나 FPS를 예상 결과로 만들어 적지 않습니다.

### 실습 31. 실제 PostgreSQL Cloud 기록과 중복 방지를 확인합니다

**EC2 SSH 터미널**에서 아래 SQL을 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform

bash scripts/11_dc.sh exec -T db sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT event_id, task, source_mode, decision FROM inference_events WHERE event_id LIKE '\''cloud:%'\'' ORDER BY id DESC LIMIT 10;"'
```

방금 분석한 이미지의 `cloud:` 이벤트가 실제로 저장되어야 합니다. 저장은 2일차 기존 테이블을 재사용합니다.

브라우저에서 `http://127.0.0.1:18000/api/v1/cloud/events`로도 기록을 확인합니다. 같은 **이벤트 ID를 중복으로 저장하지 않는 정책**은 신규 `save_cloud_event()`의 자동 테스트에서도 확인합니다. 다만 이미지 분석 버튼을 **새로 누르면 새 `event_id`로 재추론**하므로 새로운 행이 생기는 것이 정상입니다.

### 실습 32. Prometheus·Grafana에서 Cloud 측정값을 확인합니다

**PC 브라우저**에서 `http://127.0.0.1:19090/targets`를 열어 다음 Target을 확인합니다.

```text
platform_api  → api:8000       (UP)
cloud_gpu     → cloud-gpu:8100 (UP)
prometheus    → localhost:9090 (UP)
```

이어 Prometheus 쿼리를 직접 입력합니다.

```promql
up{job="cloud_gpu"}
```

```promql
sum(rate(cloud_gpu_pose_requests_total{result="ok"}[5m]))
```

```promql
1000 * histogram_quantile(
  0.95,
  sum by (le) (rate(cloud_gpu_pose_inference_seconds_bucket[5m]))
)
```

**해석:** `up=1`은 지표를 수집한다는 뜻입니다. Pose 추론 p95는 **실제 GPU 호출시간 분포**이며, 첫 이미지를 올리기 전에는 시계열이 존재하지 않을 수 있습니다. 여러 장의 이미지를 실제로 분석한 후 Grafana를 다시 확인합니다.

Grafana `http://127.0.0.1:13000`에 `.env`에서 설정한 관리자 계정으로 로그인한 뒤 **교과13 → 교과13 · 11일차 AWS GPU 모니터링** 대시보드를 엽니다. 10일차의 기존 대시보드도 남아 있는지 확인합니다.

### 실습 33. Cloud 장애 발생 시 이전 서비스 상태를 확인합니다

학습용 장애 확인은 **GPU 추론 컨테이너만 중지**하여 수행합니다. 데이터베이스나 볼륨을 삭제하지 않습니다. **EC2 SSH 터미널**에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
bash scripts/11_dc.sh stop cloud-gpu
bash scripts/11_dc.sh ps
```

Cloud GPU가 중단되면 `/api/v1/cloud/health`의 연결 상태가 `False`가 되거나 분석 API가 오류를 반환해야 합니다. 다른 FastAPI 조회 API와 PostgreSQL 데이터는 유지될 수 있습니다. **이미 저장한 기록은 삭제되지 않아야** 합니다.

이제 GPU 서비스를 다시 시작합니다.

```bash
bash scripts/11_dc.sh start cloud-gpu
bash scripts/11_dc.sh logs --tail=60 cloud-gpu
bash scripts/11_dc.sh ps
```

서비스가 다시 `healthy`인지 확인한 뒤 **허용된 이미지를 다시 분석**하고 새로운 Cloud 기록이 저장되는지 확인합니다. `start`는 새 모델이 없거나 CUDA 설정이 잘못되었을 때 이를 자동으로 고치지 않습니다. 오류 원인을 먼저 해결해야 합니다.

### 실습 34. 10일차 기능과 PostgreSQL 기록이 유지되는지 확인합니다

10일차에서 사용한 기존 경로도 그대로 확인합니다.

```text
http://127.0.0.1:18000/
http://127.0.0.1:18000/docs
http://127.0.0.1:18000/api/v1/summary
http://127.0.0.1:18000/monitoring
http://127.0.0.1:13000
```

- 기존 저장 기록을 조회하는 API가 정상인지 확인합니다.
- Grafana에서 기존 10일차 Dashboard가 삭제되지 않았는지 확인합니다.
- Redis·Celery가 실행 중인지 `bash scripts/11_dc.sh ps`로 확인합니다.
- 9일차 Jetson이 AWS에서 도달할 수 없다면 `platform_jetson_up=0`이 나타날 수 있음을 구분합니다.

Cloud의 독립적인 DB에 복원한 기록은 **백업 시점의 스냅샷**입니다. 이후 로컬 PC와 Cloud 양쪽에서 각각 새롭게 만든 기록은 별도 저장되므로 임의로 한 DB라고 생각하지 않습니다.

---

## PART 6. 자동 테스트·문제 해결·운영 문서·Git 정리

### 실습 35. 11일차 자동 테스트 파일을 작성합니다

GPU 장치가 없는 개인 PC에서도 검증할 수 있는 기능과, **실제 EC2에서만 검증되는 GPU 추론 기능**을 구분합니다.

**의사코드**

```text
Cloud JSON 계약 검증
       ↓
허위 모델/CPU 응답/다른 장치 응답 거절 확인
       ↓
PostgreSQL 테이블 저장 및 중복 방지 확인
       ↓
이미지 형식과 인증 설정 검사
       ↓
FastAPI → GPU 응답(테스트 대역) → DB 저장 검증
       ↓
Compose·모니터링 구조 검사
       ↓
실제 EC2 GPU 확인은 별도 실습 26~32로 검증
```

**파일: `tests/test_day11_cloud.py`**

```python
"""11일차: GPU/외부 AWS 없이 계약, DB, 인증, Compose 구성을 검증합니다."""
import importlib
import io
import os
from pathlib import Path
from uuid import uuid4

import pytest
import yaml
from fastapi import FastAPI
from fastapi.testclient import TestClient
from PIL import Image
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session, sessionmaker
from sqlalchemy.pool import StaticPool

os.environ.setdefault("DATABASE_URL", "sqlite+pysqlite:///:memory:")
from app import cloud_api as api
from app.cloud_repo import save_cloud_event
from app.db import Base, get_db
from app.jetson_api import PoseResponse
from app.models import CameraSource, InferenceEventRow

ROOT = Path(__file__).resolve().parents[1]
MODEL_SHA = "a" * 64


def sample_pose(**kwargs):
    base = dict(event_id=str(uuid4()), recorded_at="2026-10-17T11:00:00+00:00",
                input_ref="student.jpg", width=640, height=480, num_persons=1,
                persons=[dict(bbox_xyxy=[3, 5, 103, 205], bbox_confidence=.88,
                              keypoints=[[10., 12., .8]] * 17)],
                inference_ms=12.5, device_id="aws-ec2-gpu",
                model_id="yolo11n-pose", model_sha256=MODEL_SHA,
                runtime="pytorch_cuda", gpu_name="NVIDIA L4", note="Pose only")
    base.update(kwargs)
    return PoseResponse.model_validate(base)


@pytest.fixture
def runtime():
    engine = create_engine("sqlite+pysqlite:///:memory:",
                           connect_args={"check_same_thread": False}, poolclass=StaticPool)
    Base.metadata.create_all(engine)
    local = sessionmaker(bind=engine, expire_on_commit=False)

    def override():
        with local() as db:
            yield db

    app = FastAPI()
    app.include_router(api.router)
    app.dependency_overrides[get_db] = override
    yield TestClient(app), local
    engine.dispose()


def sample_jpg():
    b = io.BytesIO()
    Image.new("RGB", (96, 80), "white").save(b, format="JPEG")
    return b.getvalue()


def test_model_fields_success():
    item = api.pose_to_cloud_event(sample_pose())
    assert item.event_id.startswith("cloud:")
    assert item.source_mode == "live_inference"
    assert item.details["device_kind"] == "aws_ec2_gpu"
    assert item.details["num_persons"] == 1
    assert item.confidence == .88 and item.decision == "UNKNOWN"


def test_zero_people_unknown():
    item = api.pose_to_cloud_event(sample_pose(num_persons=0, persons=[]))
    assert item.confidence is None
    assert item.predicted_label is None
    assert item.reasons == ["NO_PERSON_DETECTED"]


@pytest.mark.parametrize("bad", [dict(model_id="made-up"), dict(runtime="cpu"), dict(device_id="jetson-orin-nano-super")])
def test_fake_gpu_or_wrong_device_rejected(bad):
    with pytest.raises(ValueError):
        api.pose_to_cloud_event(sample_pose(**bad))


def test_keypoints_invalid_rejected():
    with pytest.raises(ValueError):
        api.pose_to_cloud_event(sample_pose(persons=[dict(bbox_xyxy=[0,0,2,2],bbox_confidence=.9,keypoints=[[0,0,.5]])]))


def test_db_insert_and_deduplicate(runtime):
    _, local = runtime
    event = api.pose_to_cloud_event(sample_pose())
    with local() as db:
        first, inserted = save_cloud_event(db, event)
        second, repeated = save_cloud_event(db, event)
        assert inserted is True and repeated is False
        assert first.event_id == second.event_id
        assert db.query(InferenceEventRow).count() == 1
        assert db.scalar(select(CameraSource).where(CameraSource.source_key == "day11-aws-gpu-pose"))


def test_reject_wrong_db_source(runtime):
    _, local = runtime
    event = api.pose_to_cloud_event(sample_pose())
    event.event_id = "jetson:" + event.event_id
    with local() as db, pytest.raises(ValueError):
        save_cloud_event(db, event)


def test_cloud_list_and_empty(runtime):
    client, local = runtime
    assert client.get("/api/v1/cloud/events").json() == []
    item = api.pose_to_cloud_event(sample_pose())
    with local() as db:
        save_cloud_event(db, item)
    response = client.get("/api/v1/cloud/events")
    assert response.status_code == 200
    assert response.json()[0]["event_id"] == item.event_id


def test_invalid_list_limit(runtime):
    client, _ = runtime
    assert client.get("/api/v1/cloud/events?limit=0").status_code == 422


def test_missing_cloud_key_is_503(runtime, monkeypatch):
    client, _ = runtime
    monkeypatch.delenv("CLOUD_GPU_API_KEY", raising=False)
    monkeypatch.delenv("CLOUD_GPU_URL", raising=False)
    response = client.post("/api/v1/cloud/analyze", files={"image":("a.jpg",sample_jpg(),"image/jpeg")})
    assert response.status_code == 503


def test_bad_image_rejected_before_gpu(runtime, monkeypatch):
    client, _ = runtime
    monkeypatch.setenv("CLOUD_GPU_API_KEY", "x" * 32)
    monkeypatch.setenv("CLOUD_GPU_URL", "http://cloud-gpu:8100")
    response = client.post("/api/v1/cloud/analyze", files={"image":("not.jpg",b"not an image","image/jpeg")})
    assert response.status_code == 422


def test_bad_type_rejected_before_gpu(runtime, monkeypatch):
    client, _ = runtime
    monkeypatch.setenv("CLOUD_GPU_API_KEY", "x" * 32)
    monkeypatch.setenv("CLOUD_GPU_URL", "http://cloud-gpu:8100")
    response = client.post("/api/v1/cloud/analyze", files={"image":("a.txt",b"abc","text/plain")})
    assert response.status_code == 415


def test_cloud_page(runtime):
    client, _ = runtime
    assert client.get("/cloud").status_code == 200
    assert "COCO17" in client.get("/cloud").text


def test_compose_preserves_previous_services():
    base = yaml.safe_load((ROOT / "compose.yaml").read_text())
    cloud = yaml.safe_load((ROOT / "compose.cloud.yaml").read_text())
    assert all(name in base["services"] for name in ("api", "db", "worker", "redis"))
    assert "db" not in cloud["services"]
    assert "worker" not in cloud["services"]
    assert cloud["services"]["cloud-gpu"]["gpus"] == "all"
    assert "ports" not in cloud["services"]["cloud-gpu"]
    assert cloud["services"]["api"]["environment"]["CLOUD_GPU_URL"] == "http://cloud-gpu:8100"


def test_no_unverified_gpu_util_metric():
    doc = (ROOT / "monitoring/prometheus.cloud.yml").read_text()
    assert "cloud-gpu:8100" in doc
    assert "gpu_utilization_percent" not in doc


def test_gpu_failure_is_not_cpu_fallback():
    code = (ROOT / "cloud_gpu/app.py").read_text()
    assert "torch.cuda.is_available()" in code
    assert "CPU 대체 추론은 사용하지 않습니다." in code
    assert "candidate.predict" in code
    assert '"device_id": "aws-ec2-gpu"' in code


def test_all_cloud_source_files_compilable():
    import ast
    for path in [ROOT / "app/cloud_api.py", ROOT / "app/cloud_repo.py", ROOT / "cloud_gpu/app.py"]:
        ast.parse(path.read_text(encoding="utf-8"))


def test_analyze_endpoint_calls_gpu_and_persists(runtime, monkeypatch):
    client, _ = runtime
    monkeypatch.setenv("CLOUD_GPU_API_KEY", "k" * 32)
    monkeypatch.setenv("CLOUD_GPU_URL", "http://cloud-gpu:8100")
    pose = sample_pose()

    class GPUClient:
        def __init__(self, **kwargs):
            assert kwargs["trust_env"] is False
        async def __aenter__(self):
            return self
        async def __aexit__(self, *args):
            pass
        async def post(self, url, **kwargs):
            assert url == "http://cloud-gpu:8100/api/v1/cloud/pose"
            assert kwargs["headers"]["X-Cloud-Key"] == "k" * 32
            import httpx
            return httpx.Response(200, json=pose.model_dump(mode="json"),
                                  request=httpx.Request("POST", url))

    monkeypatch.setattr(api.httpx, "AsyncClient", GPUClient)
    response = client.post("/api/v1/cloud/analyze", files={"image": ("student.jpg", sample_jpg(), "image/jpeg")})
    assert response.status_code == 200, response.text
    data = response.json()
    assert data["saved"] is True
    assert data["event"]["details"]["runtime"] == "pytorch_cuda"
    assert client.get("/api/v1/cloud/events").json()[0]["event_id"] == data["event"]["event_id"]
```

1~10일차에서 사용한 **기존 Python 가상환경**에서 테스트를 실행합니다. 최소한 FastAPI, SQLAlchemy, HTTPX, Pillow, Pytest, PyYAML과 앞 일차의 코드가 준비되어 있어야 합니다. `yaml` 모듈이 없다면 다음을 추가로 설치합니다.

```bash
python -m pip install 'PyYAML>=6,<7'
python -m pytest -q tests/test_day11_cloud.py
```

정상적으로 구현한 이번 테스트 파일 기준 **19개의 테스트가 통과**해야 합니다. 학생 PC에 설치된 라이브러리 버전이 달라 실패하면 오류 내용을 확인하세요. **자동 테스트에서 모의 GPU HTTP 응답을 사용한 부분은 실제 GPU 추론 테스트가 아닙니다.** 실제 GPU의 성공은 실습 26·29·30에서 확인합니다.

또한 **기존 1~10일차 테스트 파일을 삭제하지 말고** 프로젝트의 전체 테스트를 실행해 봅니다.

```bash
python -m pytest -q
```

일부 테스트가 Raspberry Pi·Jetson 같은 실제 하드웨어를 전제로 한다면 테스트 대상과 환경을 확인하고, 단순히 통과 수치를 만들기 위해 삭제하거나 무조건 Skip 처리하지 않습니다.

### 실습 36. 11일차에 발생하기 쉬운 오류를 해결합니다

| 증상 | 먼저 확인할 원인 | 점검·해결 방법 |
|---|---|---|
| EC2 인스턴스 생성 불가 | 해당 GPU 타입의 리전 가용량·G 계열 Quota·예산 | AWS EC2 및 Service Quotas에서 확인. 승인 없이 고가 인스턴스로 변경하지 않음 |
| SSH timeout | 보안그룹 22, 공인 IP, 본인 IP 제한, SSH 키 | EC2 Connect 화면·Security Group·키 경로 확인 |
| `Permission denied (publickey)` | 키 페어/사용자 이름/권한 오류 | `ubuntu` 계정·키 페어·`chmod 400` 확인 |
| `nvidia-smi` 실패 | 잘못된 AMI/드라이버 준비 안 됨 | 지원되는 x86_64 NVIDIA GPU AMI 확인 |
| `could not select device driver` | Docker에 NVIDIA Runtime 연결 실패 | `nvidia-ctk`·Container Toolkit 확인 |
| `CUDA: False` | CUDA Runtime/GPU Driver 호환성 오류 | 호스트 `nvidia-smi`→GPU Docker 테스트→PyTorch 순서로 확인 |
| `No such file: yolo11n-pose.pt` | 모델 복사가 빠짐·잘못된 경로 | `artifacts/day05/jetson/`와 `/models/` 마운트 및 SHA256 확인 |
| GPU 컨테이너 재시작 반복 | 모델 로드·가중치·메모리 오류 | `bash scripts/11_dc.sh logs --tail=100 cloud-gpu` |
| EC2 웹이 안 열림 | SSH 터널 중지·로컬 포트 충돌 | 터널 터미널 유지, EC2 Docker ps 및 127.0.0.1 포트 확인 |
| DB에 이벤트가 안 보임 | Cloud API 오류·DB 복원 실패 | `/api/v1/cloud/analyze` HTTP 응답·`cloud_repo.py`·DB SQL 확인 |
| `relation inference_events does not exist` | DB 백업 미복원·테이블 미생성 | 빈 DB에만 pg_restore, 또는 2일차 테이블 생성 코드 확인 |
| Compose 환경변수 설정 오류 | `.env` 누락 또는 키 값 미입력 | `chmod 600 .env`, 필수 항목 확인. 비밀값 전체 출력 금지 |
| Grafana Cloud 그래프가 비어 있음 | 실제 추론 요청 없음·Prometheus 대상 DOWN | 먼저 이미지 실제 분석, `/targets`와 PromQL 확인 |
| `platform_jetson_up=0` | 사설망 Jetson에 AWS에서 접근 불가 | 오늘의 Cloud GPU 건강 상태와 분리하여 해석 |
| `No space left on device` | CUDA 이미지·PyTorch·EBS 공간 부족 | `df -h`, Docker 사용량 확인, 승인된 EBS 용량 조정 |

**오류 점검 순서:** EC2 인스턴스 → SSH → `nvidia-smi` → Docker GPU → 실제 모델 파일 → Compose → API → PostgreSQL → 웹 화면 → Prometheus/Grafana 순서로 확인합니다. 인스턴스 전체를 무작정 다시 만드는 것보다 실패 단계부터 수정합니다.

### 실습 37. 운영 기록과 12일차 인수 문서를 작성합니다

**파일: `docs/day11_cloud_runbook.md`**를 생성해 다음 형식으로 작성하고, 실제 결과로 빈칸을 채웁니다.

~~~markdown
# 교과 13 · 11일차 AWS Cloud GPU 운영 기록

## 1. EC2 실행환경
- AWS Region:
- EC2 인스턴스 이름 및 인스턴스 유형:
- 선택한 x86_64 GPU AMI:
- Ubuntu 버전:
- `nvidia-smi`의 GPU 이름/Driver Version:
- Docker/Compose 버전:
- 실행 시작 시각/종료 시각:
- 비용 확인 방법:

## 2. 실제 파일·데이터 인수
- 모델 경로: `artifacts/day05/jetson/yolo11n-pose.pt`
- 모델 SHA256 (PC / Cloud 동일 여부):
- PC 기존 DB 건수:
- Cloud 복원 직후 DB 건수:
- Cloud 새 추론 건수:

## 3. 실행 확인
- Docker 7개 서비스 상태:
- Cloud `/health` GPU 사용 가능 여부:
- `/cloud` 화면에서 실제 사람 검출 여부:
- `inference_ms` 실측값:
- `event_id` cloud: 형식 및 PostgreSQL 저장 여부:

## 4. 모니터링
- Prometheus `platform_api`, `cloud_gpu` Target 상태:
- Grafana 11일차 Dashboard 확인 결과:
- 이미지 업로드 전/후 메트릭 변화:

## 5. 장애와 복구
- 중단한 컨테이너:
- 관찰한 HTTP 오류·Prometheus 변화:
- 복구 명령과 확인 결과:

## 6. 보안·비용·종료
- Security Group SSH/IP 제한 확인:
- 8000/3000/9090/8100 공개 여부(모두 공개하지 않음):
- 백업 및 키 보관 위치(비밀값 자체는 기록하지 않음):
- 종료 전 추가 Cloud DB 백업 여부:
- EC2 Stop/Terminate 및 EBS·IPv4·스냅샷 확인:

## 7. 12일차 연결
- Git에 기록한 코드 버전:
- EC2 배포를 다시 실행할 명령:
- 배포 시 자동 테스트·Rollback에 사용할 백업/태그:
~~~

README와 `docs/architecture.md`에도 **11일차에 추가한 Cloud GPU 서비스와 실제 접속 방법**을 정리합니다.

README에 이어 붙일 내용 예시:

~~~markdown
## 11일차 — AWS EC2 GPU Pose 배포

- 기존 7일차 Compose·10일차 모니터링을 유지합니다.
- `compose.cloud.yaml`에서 Cloud GPU 추론 컨테이너를 추가합니다.
- EC2에서 `bash scripts/11_dc.sh up -d`로 실행합니다.
- 모델 가중치는 `artifacts/day05/jetson/yolo11n-pose.pt`를 사용합니다.
- Web: SSH 터널을 연 PC에서 `http://127.0.0.1:18000/cloud`
- Grafana: SSH 터널을 연 PC에서 `http://127.0.0.1:13000`
- `/api/v1/cloud/analyze`는 실제 YOLO11n-Pose CUDA 추론을 수행합니다.
- DB 결과는 `cloud:` 접두사와 `source_mode=live_inference`를 사용합니다.
- 이전 PC PostgreSQL과 AWS DB는 자동 동기화되지 않습니다.
- 12일차에는 동일 Cloud 구성에 CI/CD와 Rollback을 추가합니다.
~~~

### 실습 38. 실습 종료 전에 Cloud DB를 백업하고 비용을 정리합니다

11일차에 Cloud PostgreSQL에 새로 기록된 이벤트는 원래 PC DB에는 없을 수 있습니다. EC2를 종료하거나 제거하기 전에 **새 기록까지 포함한 Cloud DB 백업**을 만듭니다.

**EC2 SSH 터미널**에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
mkdir -p backups/day11

bash scripts/11_dc.sh exec -T db sh -c \
  'pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc' \
  > backups/day11/cloud_after_class.dump

ls -lh backups/day11/cloud_after_class.dump
```

이 백업이 필요하다면 기관에서 승인한 보관 경로로 옮깁니다. 절대 Git으로 올리지 않습니다. Cloud 로그, 모델, 원본 데이터, 백업 보존 정책에 따라 종료 여부를 결정합니다.

AWS Console에서 실제 작업을 마친 후 **EC2를 Stop 또는 Terminate**합니다. 12일차 수업에서 같은 인스턴스를 사용할 계획이라면 **Stop 후 EBS 유지 비용을 확인**합니다. 인스턴스를 더 이상 쓰지 않으면 필요한 백업을 확보한 뒤 **Terminate, EBS·스냅샷·공인 IPv4 등 잔여 리소스를 점검**합니다. 단, 원본 데이터를 삭제하는 작업은 허가 없이 수행하지 마세요.

**비용 관련 중요 사항:** `docker compose stop`이나 SSH 로그아웃만으로 EC2 인스턴스 요금이 중단되지 않습니다. **AWS EC2 인스턴스 상태를 별도로 확인**해야 합니다. Stop만 하더라도 EBS 비용은 남을 수 있습니다.

### 실습 39. Git 업로드 제외 규칙과 최종 커밋을 확인합니다

먼저 7~10일차에 만든 `.gitignore`를 유지하고 다음 항목이 제외되는지 확인합니다. `.env`, `.pem`, DB 백업, 모델·데이터가 Git에 포함되어서는 안 됩니다.

```gitignore
# 11일차 클라우드 배포 파일 중 비공개 자료
.env
.env.*
*.pem
*.key
backups/
artifacts/
handoff/records/
handoff/provided/
data/day03/exports/
```

이미 추적 중인 파일은 `.gitignore`를 추가해도 자동으로 제거되지 않습니다. `git status` 및 `git ls-files`로 확인하세요.

**기존 PC 프로젝트**에서 소스·설정만 기록합니다.

```bash
cd ~/ai_vision/subject13_ai_platform

git status

git add app/cloud_api.py app/cloud_repo.py app/main.py \
  cloud_gpu/app.py cloud_gpu/Dockerfile.cloud-gpu \
  compose.cloud.yaml compose.cloud.monitoring.yaml \
  monitoring/prometheus.cloud.yml \
  monitoring/grafana/dashboards/subject13_day11_cloud.json \
  web/cloud.html scripts/11_dc.sh scripts/11_check_cloud_inputs.py \
  tests/test_day11_cloud.py docs/day11_cloud_runbook.md \
  README.md docs/architecture.md .gitignore

git status
```

스테이징 목록에 `.env`·모델·백업·허가되지 않은 이미지 파일이 없는지 확인한 뒤 커밋합니다.

```bash
git commit -m "교과13 11일차: AWS EC2 GPU Pose 배포와 Cloud 모니터링"
git log -1 --oneline
git status
```

기관에서 외부 GitHub 업로드를 허용한 경우에만 아래 명령으로 원격 저장소에 업로드합니다.

```bash
git push origin main
```

폐쇄망 환경이거나 외부 서비스에 소스코드를 올릴 수 없다면 로컬 Git 또는 승인된 내부 Git 저장소를 사용합니다.

---

# 6. 오늘 작성한 내용의 최종 정리 및 GitHub 업로드

## 6.1. 오늘 완성한 기능

```text
10일차 중앙 플랫폼 (FastAPI/DB/Redis/Celery/Prometheus/Grafana)
                           ↓ 코드 재사용 · Cloud 배포
                       AWS EC2 GPU
                           ↓
       실제 YOLO11n-Pose CUDA 이미지 추론 (pytorch_cuda)
                           ↓
               BBox · COCO17 Keypoint · Confidence
                           ↓
           InferenceEvent(event_id=cloud:..., decision=UNKNOWN)
                           ↓
                    기존 PostgreSQL 테이블 저장
                           ↓
           /cloud 웹 화면 · Prometheus · Grafana
                           ↓
              장애·DB 백업·접근제어·비용 종료 확인
```

7일차의 기존 컨테이너 4개와 10일차 모니터링 2개에 **11일차 Cloud GPU 컨테이너 1개**를 추가했습니다. Jetson TensorRT `.engine`을 EC2에 복사하지 않고, 기존 YOLO11n-Pose 가중치를 **x86_64 CUDA 컨테이너**에서 실행하는 방법을 배웠습니다.

**실제 모델 실행 결과를 성공이라고 판단하는 기준**은 다음과 같습니다. `nvidia-smi`와 Docker GPU 검증을 통과했고, `torch.cuda.is_available()==True`가 확인되었으며, 본인이 업로드한 이미지가 실제 모델을 거쳐 Bounding Box·Keypoint·새 `event_id`를 반환하고, 그 이벤트가 PostgreSQL에 저장되어야 합니다. 단순한 `/health=200`이나 자동 테스트만으로 모든 검증이 끝난 것은 아닙니다.

## 6.2. 오늘의 자가 체크리스트

- [ ] 기존 10일차 `subject13_ai_platform`을 새로 만들지 않고 이어서 사용했습니다.
- [ ] PC PostgreSQL 백업 파일을 만들고, Cloud DB에 복원한 개수를 확인했습니다.
- [ ] AWS GPU 인스턴스의 비용과 보안 그룹·GPU 할당량을 확인했습니다.
- [ ] EC2에서 `uname -m=x86_64`, `nvidia-smi` 및 Docker CUDA 실행을 확인했습니다.
- [ ] 교과 10의 실제 `yolo11n-pose.pt`를 별도 전송하고 SHA256을 대조했습니다.
- [ ] 기존 Compose 파일을 지우지 않고 Cloud용 두 Compose 오버레이를 추가했습니다.
- [ ] `cloud-gpu`를 외부 포트 없이 Docker 내부 네트워크에서 실행했습니다.
- [ ] SSH 터널로 `/cloud`, Grafana, Prometheus에 접속했습니다.
- [ ] 실제 JPG/PNG 한 장 이상을 AWS CUDA GPU에서 추론했습니다.
- [ ] `cloud:` 이벤트가 PostgreSQL에 새로 저장되었고 `decision=UNKNOWN`의 이유를 설명할 수 있습니다.
- [ ] Grafana에서 실제 Cloud 추론 지표를 확인했습니다.
- [ ] 기존 10일차 API와 대시보드가 삭제되지 않았습니다.
- [ ] `python -m pytest -q tests/test_day11_cloud.py`에서 테스트를 실행했습니다.
- [ ] 실습 종료 후 Cloud DB 백업과 EC2 과금 상태를 확인했습니다.
- [ ] 소스코드만 Git에 커밋하고 모델·DB·인증키를 올리지 않았습니다.

## 6.3. 12일차와의 연결

12일차에는 **이번에 만든 Cloud Compose 실행 방식을 그대로 사용하면서**, Git의 코드 변경을 자동 테스트·Docker 이미지 빌드·배포 단계로 연결합니다. 문제가 생겼을 때는 이전 정상 버전으로 돌아가는 **Rollback**을 실습합니다. 따라서 오늘의 `docs/day11_cloud_runbook.md`, 코드 커밋, 안전하게 보관한 DB 백업이 12일차의 출발점이 됩니다.

---

## 참고할 공식 문서

아래 공식 문서는 AWS 리전별 AMI·인스턴스 종류와 NVIDIA 컨테이너 동작처럼 바뀔 수 있는 환경 설정을 확인할 때 사용합니다.

- [AWS Deep Learning Base GPU AMI (Ubuntu 24.04)](https://docs.aws.amazon.com/dlami/latest/devguide/aws-deep-learning-x86-base-gpu-ami-ubuntu-24-04.html)
- [Amazon EC2 Accelerated Computing Instance Specifications](https://docs.aws.amazon.com/ec2/latest/instancetypes/ac.html)
- [NVIDIA Container Toolkit Docker GPU 테스트](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/sample-workload.html)
- [AWS EC2 인스턴스 상태와 요금](https://docs.aws.amazon.com/us_en/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html)
- [중지된 EC2와 EBS 요금 안내](https://repost.aws/knowledge-center/ebs-charge-stopped-instance)

> **검증 범위:** 본 수업의 Cloud 연결 코드·기존 DB 저장·자동 테스트·Compose 설정은 검증용 로컬 환경에서 확인했습니다. 그러나 실제 AWS 인스턴스 생성, 유료 GPU 실행, EBS 복원, 실제 GPU FPS·모델 추론 및 네트워크 환경은 사용자 계정의 AWS 실습 장비에서 진행해야 합니다. 시험용 대역에서 통과한 결과를 실제 GPU 실측으로 표기하지 않습니다.

