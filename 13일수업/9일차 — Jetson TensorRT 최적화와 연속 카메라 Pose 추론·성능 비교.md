## 1. 오늘의 목표

8일차에는 교과 10의 실제 `yolo11n-pose.pt`를 Jetson GPU Docker에서 실행하고, **이미지 한 장의 사람 Bounding Box와 COCO 17 Keypoint**를 PC FastAPI에 전달했습니다. 그 결과를 2일차 PostgreSQL에 `jetson:` 이벤트로 저장하고 웹 화면의 Skeleton으로 확인했습니다.

오늘은 **바로 그 YOLO11n-Pose 모델**을 재사용하여 Jetson GPU 추론 실행 방식을 TensorRT로 최적화하고, 정지 이미지에서 **카메라 연속 프레임**으로 입력을 확장합니다. 같은 `subject13_ai_platform` 프로젝트에 새 파일을 추가하며, 7일차 PC Docker Compose와 기존 PostgreSQL 데이터는 그대로 유지합니다.

수업을 마치면 다음을 직접 수행할 수 있습니다.

1. 5일차 PT·ONNX·Manifest, 8일차 Jetson Pose 컨테이너를 확인합니다.
2. TensorRT 엔진이 GPU·JetPack 환경과 연결되는 이유를 설명합니다.
3. **Jetson 장치에서** 실제 YOLO11n-Pose 모델을 TensorRT FP16 `.engine`으로 내보내고 SHA256을 검증합니다.
4. 같은 실제 이미지·입력 크기·반복 횟수에서 PyTorch CUDA와 TensorRT를 측정하여 p50·p95 지연시간과 순차 처리 FPS를 비교합니다.
5. IP Webcam 또는 USB 카메라에서 연속 프레임을 읽고 Jetson TensorRT로 Pose를 추론합니다.
6. PC 브라우저에서 최신 프레임에 Bounding Box·Skeleton을 그린 뒤 추론시간·실측 처리 FPS를 확인합니다.
7. 원하는 순간의 **최신 Pose 이벤트 한 건**을 기존 PostgreSQL에 저장하고 중복 저장을 확인합니다.
8. 네트워크 또는 카메라 장애 때 상태가 표시되고 연결 복구 뒤 화면이 갱신되는지 확인합니다.
9. 기존 8일차 프로그램과 새 코드를 자동 테스트하고 README·Git에 작업을 기록합니다.

**오늘의 완성 기준:** Jetson에서 생성한 엔진으로 실제 카메라 프레임을 연속 분석하고, PC `/jetson-stream`에서 Skeleton과 측정값이 보이며, 저장 버튼으로 기존 `inference_events` 테이블에 `runtime=tensorrt` 이벤트가 기록되어야 합니다. **TensorRT 엔진을 실제 장치에서 빌드하지 않았다면 최종 완성으로 보지 않습니다.**

---

## 2. 오늘 수업의 전체 흐름

![[Pasted image 20261018131626.png]]

```text
[1~8일차 기존 프로젝트를 그대로 열기]
PC: FastAPI + PostgreSQL + Redis + Celery Compose
Jetson: JetPack 6 + GPU Docker + day08 Pose 이미지 API
                          ↓
[5일차 실제 모델 확인]
yolo11n-pose.pt / yolo11n-pose.onnx / Manifest
                          ↓
[Jetson GPU에서 TensorRT 엔진 생성]
같은 PT → 내부 ONNX 변환 → TensorRT FP16 engine
                          ↓
[엔진 SHA256 · 실행 검증]
                          ↓
[동일 이미지 성능 비교]
PyTorch CUDA(.pt) ↔ TensorRT(.engine)
p50·p95 추론시간 / 순차 처리 FPS / 사람 검출 수
                          ↓
[Jetson 카메라 입력 연결]
IP Webcam 또는 USB → OpenCV Frame
                          ↓
[TensorRT YOLO11n-Pose]
Person BBox + COCO 17 Keypoint + Confidence
                          ↓
[Jetson 최신 프레임 API]
이미지 JPEG + Pose JSON + 실제 처리 FPS
                          ↓
[PC 기존 FastAPI]
카메라 최신 프레임 중계 · 공통 JSON 검증
                          ↓
[PC 브라우저 /jetson-stream]
0.5초 주기 갱신 + BBox·Skeleton 표시
                          ↓
[선택한 최신 이벤트 저장]
기존 PostgreSQL inference_events
                          ↓
[연결 끊김 · 복구 시험 → 자동 테스트 → README·Git]
                          ↓
[10일차] 지금 측정하는 FPS·추론시간·장애 상태를 모니터링 지표로 확장
```

**오늘도 교과 10의 Random Forest로 6개 Action을 분류하지 않습니다.** Jetson GPU 모델은 **Pose 검출**이며, 사람을 찾았거나 못 찾은 사실만으로 위험행동·정상작업을 판정하지 않습니다. 공통 이벤트의 `decision`은 `UNKNOWN`으로 유지합니다.

---

## 3. 실습 전에 알아야 할 핵심

### 3.1. 오늘 1~8일차 중 어떤 파일을 이어서 사용하나요?

| 이전 실습 | 기존 결과물 | 9일차에서 하는 일 |
|---|---|---|
| 1일차 | `app/schemas.py`의 `InferenceEvent` | 공통 이벤트 구조 유지 |
| 2일차 | `app/db.py`, `app/models.py`, `inference_events` | 같은 DB와 테이블에 저장 |
| 3일차 | Redis·Celery 작업 | 그대로 실행·유지, 9일차에 변경 없음 |
| 4일차 | 실제 OCR·Action 서비스 | 변경하지 않음 |
| 5일차 | `artifacts/day05/jetson/yolo11n-pose.pt`, `.onnx`, `pose_export_manifest.json` | 모델 검증 및 엔진 생성 |
| 6일차 | Raspberry Pi·GPIO | 변경하지 않음 |
| 7일차 | PC `compose.yaml`, `Dockerfile` | 서버·DB 실행 환경 그대로 유지 |
| 8일차 | `app/jetson_api.py`, `app/jetson_repo.py`, `deploy/day08_jetson/` | TensorRT 응답을 수용하고 Jetson 서버 확장 |

8일차에 사람이 있는 정지 사진으로 실습했다면 **그 이미지 한 장을 성능 비교에도 그대로** 사용합니다. 9일차 성능 측정이 끝난 뒤 카메라를 연결하므로, 어느 단계에서 오류가 생겼는지 구별할 수 있습니다.

### 3.2. ONNX와 TensorRT 엔진은 어떻게 연결되나요?

```text
PyTorch 모델 (.pt)
      ↓ 모델 계산 구조 내보내기
ONNX 계산 그래프 (.onnx)
      ↓ Jetson GPU용 TensorRT 빌더
TensorRT 엔진 (.engine)
      ↓
Jetson GPU 최적화 추론
```

5일차에 저장한 `yolo11n-pose.onnx`는 **모델을 ONNX로 내보내는 과정을 검증한 기존 산출물**입니다. 이번 실습에서 사용하는 `YOLO(...).export(format="engine")`는 **같은 `.pt`에서 내부적으로 ONNX 중간 그래프를 다시 만들어 TensorRT 엔진을 빌드**합니다. 5일차의 ONNX 파일 자체를 직접 TensorRT 엔진으로 변환했다고 표현하지 않습니다. 두 기존 모델 파일의 SHA256은 모두 기록하며, 어떤 PT로 엔진을 만들었는지 추적합니다.

TensorRT 엔진은 Jetson GPU, TensorRT 버전 및 빌드 환경에 영향을 받습니다. **PC에서 `.engine`을 임의로 만들어 Jetson에 복사하지 않고, 오늘 사용할 Jetson 장치에서 엔진을 생성**합니다.

### 3.3. FP16은 무엇인가요?

FP32와 FP16은 연산에서 사용하는 수치 표현 정밀도의 차이입니다. FP16은 메모리·연산에 유리할 수 있으나, **속도가 반드시 빨라지거나 검출 결과가 완전히 같다고 가정할 수 없습니다.** 오늘은 실제 이미지에 대해 사람 수·BBox·Keypoint를 비교하고 성능 수치를 측정해 판단합니다. INT8은 별도 교정 데이터와 검증이 필요하므로 오늘 사용하지 않습니다.

### 3.4. FPS를 세 가지로 구분합니다

| 이름 | 무엇을 측정하나요? | 오늘 확인 위치 |
|---|---|---|
| **순차 추론 FPS 추정치** | 준비된 한 이미지를 순서대로 반복해 모델을 호출한 처리량 | `benchmark.csv` |
| **연속 처리 FPS** | Jetson에서 실제 카메라 프레임 추론이 끝난 간격 기준 | `Jetson /latest`, 웹 화면 |
| **브라우저 갱신률** | PC 브라우저가 0.5초마다 최신 프레임을 조회하는 빈도 | 웹 JavaScript (약 2회/초) |

브라우저가 초당 두 번 갱신한다고 해서 **Jetson 모델 FPS가 2인 것은 아닙니다.** 네트워크 전송·웹 그리기·DB 저장은 벤치마크의 모델 지연시간과 구분합니다.

### 3.5. 연속 카메라에서 중요한 점

카메라 입력 → 모델 추론 → 결과 표시의 속도가 서로 다를 수 있습니다. 여기서는 **카메라가 열려 있는 동안 최신 프레임을 반복 처리하고, 화면에는 가장 최근의 완성된 프레임만 전달**합니다. 모든 카메라 프레임을 장기 보관하거나 큐에 쌓지 않습니다. 카메라 연결이 끊기면 이전 프레임을 최신 화면으로 계속 표시하지 않고 재연결 상태로 변경합니다.

**오늘의 운영 한계:** 단일 Jetson·단일 카메라·단일 GPU 추론 Worker로 구성합니다. 이 코드는 영상의 모든 프레임을 빠짐없이 분석하는 보증 시스템이나 완성된 산업안전 경고장치가 아닙니다.

### 3.6. 이전 결과물이 없는 경우

| 누락된 항목 | 오늘 필요한 최소 제공 자료 |
|---|---|
| 5일차 모델 준비 실패 | 실제 `yolo11n-pose.pt`, `yolo11n-pose.onnx`, `pose_export_manifest.json` |
| 8일차 Jetson 배포 미완성 | 8일차 `deploy/day08_jetson/` 소스 및 `.pt` 모델, Jetson Docker 실행 설정(장치별 값은 새로 설정) |
| 8일차 사진이 없음 | 촬영·공유가 허락된 사람 이미지 JPG/PNG 1장 |
| IP Webcam 입력 없음 | 허가된 강의실 USB 카메라 1대 또는 접근 가능한 IP Webcam 영상 주소 |
| 7일차 PC 플랫폼 미완성 | 기존 교과 13의 실제 프로젝트 코드·DB 스키마·실행환경 복구 |

앞 교과의 전체 ZIP을 다시 다운로드하지 않습니다. **이미 생성해 둔 프로젝트와 산출물을 그대로 사용**합니다. 다른 학생의 모델 파일을 제공받았다면 파일명뿐 아니라 모델 출처와 SHA256을 확인합니다. 장치 인증 키와 비밀번호는 제공받은 소스에 포함하지 않습니다.

### 3.7. 오늘 사용할 컴퓨터와 터미널을 구분합니다

```text
[PC WSL/Ubuntu]  ~/ai_vision/subject13_ai_platform
                 소스 편집 · Git · PC Docker Compose

[Jetson SSH]      ~/ai_vision/subject13_ai_platform
                 TensorRT 변환 · GPU Docker · 카메라 접근

[PC 브라우저]      http://127.0.0.1:8000/jetson-stream
```

**Jetson의 `127.0.0.1`과 PC의 `127.0.0.1`은 다릅니다.** 서버 주소·모델 파일 경로·볼륨 경로를 실행하는 장치별로 구분해야 합니다. 아직 네트워크에 연결되지 않은 Jetson의 IP를 임의로 추측하지 않습니다.

---

## 4. 오늘 사용할 기술 스택

| 기술 | 오늘 사용하는 기능 |
|---|---|
| NVIDIA Jetson Orin Nano Super / JetPack 6 | 실제 ARM64 GPU 최적화·연속 카메라 처리 |
| Ultralytics YOLO11n-Pose | 기존 17-Keypoint 모델 읽기·TensorRT 내보내기·추론 |
| ONNX / TensorRT FP16 | 모델 중간 형식 및 Jetson 장치 최적화 엔진 |
| PyTorch CUDA | 8일차 GPU 실행 성능 비교 기준 |
| OpenCV / `VideoCapture` | IP Webcam 또는 USB 카메라에서 실제 프레임 읽기 |
| Docker / Compose / NVIDIA Runtime | Jetson의 GPU 모델 서버 실행 |
| Python `threading` / `deque` | 카메라 연속 처리·최근 FPS 계산 |
| FastAPI / HTTPX / Pydantic | Jetson 프레임 중계 및 응답 검증 |
| HTML Canvas / JavaScript | 최신 이미지 위에 Skeleton과 BBox 표시 |
| SQLAlchemy / PostgreSQL | 기존 `inference_events`에 Pose 기록 저장 |
| Pytest / Git | 자동 검증·일차별 코드 이력 기록 |

**필수 전제:** 8일차 실습을 통과한 JetPack 6 Jetson과 PC Docker Compose가 있어야 합니다. Jetson의 실제 TensorRT 버전·GPU 성능·카메라 주소는 본인의 장치에서 확인합니다.

---

# 5. 처음부터 끝까지 따라 하는 상세 실습

## PART 1. 8일차 프로젝트와 오늘의 입력을 확인합니다

### 실습 1. 기존 교과 13 프로젝트를 엽니다

**PC WSL/Ubuntu 터미널**에서 8일차까지 사용한 폴더를 그대로 엽니다.

```bash
cd ~/ai_vision/subject13_ai_platform
pwd
ls -lh app/main.py app/jetson_api.py app/jetson_repo.py   deploy/day08_jetson/app_jetson.py deploy/day08_jetson/compose.jetson.yaml

docker compose ps
```

`api`, `db`, `redis`, `worker`가 실행 중인지 확인합니다. 중단되어 있다면 기존 환경에서 아래처럼 시작합니다.

```bash
docker compose up -d
docker compose ps
```

기존 PostgreSQL 기록을 보존해야 하므로 **`docker compose down -v`는 실행하지 않습니다.** 오늘은 DB 테이블을 새로 만들거나 데이터베이스를 초기화하지 않습니다.

### 실습 2. 5일차 모델과 8일차 시험 이미지를 확인합니다

**PC 터미널**에서 확인합니다.

```bash
ls -lh artifacts/day05/jetson/yolo11n-pose.pt   artifacts/day05/jetson/yolo11n-pose.onnx   artifacts/day05/jetson/pose_export_manifest.json   deploy/day08_jetson/samples/test_pose.jpg
sha256sum artifacts/day05/jetson/yolo11n-pose.pt \
  artifacts/day05/jetson/yolo11n-pose.onnx
```

파일이 없다면 해당 5·8일차 산출물부터 복구합니다. 서로 다른 모델을 같은 이름으로 바꾸지 않습니다. 사람이 나온 **한 장의 동일한 테스트 이미지**를 비교에 사용해야 결과 차이를 확인할 수 있습니다.

### 실습 3. Jetson의 현재 추론 서버를 확인합니다

**Jetson SSH 터미널**을 열고 실제 장치 폴더로 이동합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
uname -m
cat /etc/nv_tegra_release
sudo docker info --format '{{json .Runtimes}}'
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml ps
```

`aarch64`, JetPack 6 계열의 L4T, `nvidia` Runtime을 확인합니다. 8일차 `/health`가 준비 상태인지 확인합니다. 아래 `JETSON_HOST_IP`에는 **Jetson에서 확인한 LAN IP**를 설정합니다.

```bash
export JETSON_HOST_IP=실제_Jetson_LAN_IP
curl --fail "http://${JETSON_HOST_IP}:8018/health"
```

8일차 GPU 서버가 정상이라면 다음으로 진행합니다. **아직 기존 컨테이너를 중단하지 않습니다.**

### 실습 4. Jetson TensorRT Python 패키지를 확인합니다

8일차 Jetson GPU 이미지 안에서 CUDA·TensorRT·Ultralytics가 실제로 인식되는지 확인합니다. **Jetson 터미널**에서 실행합니다.

```bash
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml exec pose   python3 -c 'import torch, ultralytics; print("CUDA", torch.cuda.is_available()); print("Ultralytics", ultralytics.__version__)'

docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml exec pose   python3 -c 'import tensorrt as trt; print("TensorRT", trt.__version__)'
```

`CUDA True`와 TensorRT 버전이 출력되는지 확인합니다. `No module named tensorrt`가 발생하면 **JetPack에 맞는 TensorRT Python 바인딩이 설치된 컨테이너 이미지인지 확인**합니다. 일반 PC의 TensorRT 패키지를 무조건 설치하지 않습니다. JetPack마다 호환되는 TensorRT 버전이 다르므로 장치 제공 환경을 우선 확인합니다.

---

## PART 2. TensorRT 엔진 생성과 실제 성능 비교를 준비합니다

### 실습 5. 9일차 폴더를 만듭니다

다시 **PC 터미널**로 이동합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
mkdir -p artifacts/day09/jetson artifacts/day09/benchmarks
mkdir -p deploy/day08_jetson/samples
ls -ld deploy/day08_jetson artifacts/day09/jetson
```

**새로운 교과 13 프로젝트를 생성하지 않습니다.** 8일차 `deploy/day08_jetson/`에 파일을 추가하고, 실제 Jetson에서 만든 엔진만 새 `artifacts/day09/jetson/`에 저장합니다. `.engine`은 특정 장치에서 만든 실행 바이너리이므로 원본 `.pt`와 별도로 보관합니다.

### 실습 6. 카메라 입력과 FPS 계산에 사용하는 코드를 작성합니다

**왜 필요한가요?** 다음 카메라 코드는 이미지 한 장이 아니라 연속 프레임을 처리하므로, 카메라 URL·장치 번호를 구분하고 실제 완료 간격에서 FPS를 계산해야 합니다.

**의사코드**

```text
카메라 설정값 읽기
  ├─ "0"처럼 숫자면 USB 카메라 장치 번호
  ├─ http(s)/rtsp 주소면 IP Webcam
  └─ 그 외는 잘못된 입력으로 거절
                ↓
최근 프레임 처리 완료 시각 30개 기록
                ↓
(최근 프레임 간격 수) / (경과시간) = 실측 처리 FPS
```

**파일: `deploy/day08_jetson/day09_utils.py`**

```python
"""9일차: Jetson 카메라 입력·FPS 집계의 하드웨어 독립 부분."""
from collections import deque
from time import monotonic


def parse_camera_source(value: str):
    value = value.strip()
    if not value:
        raise ValueError("DAY09_CAMERA_SOURCE가 비어 있습니다.")
    if value.isdecimal():
        return int(value)
    if value.startswith(("http://", "https://", "rtsp://")):
        return value
    raise ValueError("카메라 주소는 0 같은 정수 또는 http(s)/rtsp URL만 허용합니다.")


class FrameRate:
    def __init__(self, window: int = 30):
        self.times = deque(maxlen=window)

    def tick(self, now: float | None = None) -> float:
        self.times.append(monotonic() if now is None else now)
        if len(self.times) < 2:
            return 0.0
        duration = self.times[-1] - self.times[0]
        return round((len(self.times) - 1) / duration, 2) if duration > 0 else 0.0
```

`parse_camera_source()`가 `0`을 정수로 변환해야 OpenCV가 USB 카메라 장치 번호로 인식합니다. `FrameRate.tick()`은 추론을 **완료한 간격**에서 FPS를 계산하고, 처음 한 프레임만 있을 때는 0을 반환합니다.

### 실습 7. TensorRT 엔진을 만드는 코드를 작성합니다

이 코드에서는 **5일차의 실제 두 모델 파일이 있는지 확인하고**, Jetson 컨테이너의 CUDA 상태를 검사한 뒤 FP16 TensorRT 엔진을 만듭니다. PT 원본은 읽기 전용 볼륨이므로 **엔진 생성용 PT 복사본을 쓰기 가능한 `artifacts/day09/jetson/build/`에 만듭니다.** 기존 5일차 파일은 수정하지 않습니다.

**의사코드**

```text
실행 장치 = Jetson aarch64 확인
5일차 PT·ONNX 실제 파일 확인
CUDA 사용 가능 / TensorRT Python 바인딩 확인
                ↓
PT 원본 → 쓰기 가능한 build 폴더에 복사
                ↓
YOLO.export(format="engine", FP16, imgsz=640, batch=1)
                ↓
GPU에서 TensorRT 엔진 Warm-up·Pose 출력 확인
                ↓
.engine SHA256 기록
                ↓
engine_manifest.json 저장
```

엔진 빌드 과정에서 오류가 발생하면 `.pt` CUDA 추론으로 조용히 대신 실행하지 않습니다. 오류 원인을 확인한 후 엔진을 다시 생성합니다.

**파일: `deploy/day08_jetson/export_tensorrt.py`**

```python
"""Jetson 장치 안에서 같은 YOLO11n-Pose .pt를 TensorRT FP16으로 내보냅니다."""
import argparse
import hashlib
import json
import platform
import shutil
from datetime import datetime, timezone
from pathlib import Path


def digest(path: Path) -> str:
    h = hashlib.sha256()
    with path.open("rb") as f:
        for chunk in iter(lambda: f.read(1048576), b""):
            h.update(chunk)
    return h.hexdigest()


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--pt", type=Path, required=True)
    parser.add_argument("--onnx", type=Path, required=True)
    parser.add_argument("--out", type=Path, required=True)
    parser.add_argument("--imgsz", type=int, default=640)
    args = parser.parse_args()
    if platform.machine() != "aarch64":
        raise RuntimeError("이 실습의 엔진은 Jetson ARM64에서 생성합니다.")
    for path in (args.pt, args.onnx):
        if not path.is_file() or path.stat().st_size == 0:
            raise FileNotFoundError(f"5일차 모델 파일이 없습니다: {path}")
    import torch
    import ultralytics
    import tensorrt as trt
    from ultralytics import YOLO
    if not torch.cuda.is_available():
        raise RuntimeError("Jetson CUDA를 사용할 수 없습니다. TensorRT 엔진을 생성하지 않습니다.")
    args.out.parent.mkdir(parents=True, exist_ok=True)
    build_dir = args.out.parent / "build"
    build_dir.mkdir(exist_ok=True)
    pt_copy = build_dir / args.pt.name
    shutil.copy2(args.pt, pt_copy)
    model = YOLO(str(pt_copy))
    settings = dict(format="engine", imgsz=args.imgsz, batch=1,
                    dynamic=False, device=0, workspace=1)
    try:
        output = model.export(**settings, quantize=16)
        precision_arg = "quantize=16"
    except (TypeError, SyntaxError) as exc:
        # 구버전 Ultralytics만 half=True를 사용합니다. FP32로 조용히 전환하지 않습니다.
        if "quantize" not in str(exc).lower():
            raise
        output = model.export(**settings, half=True)
        precision_arg = "half=True (legacy)"
    generated = Path(output)
    if not generated.is_file() or generated.suffix != ".engine" or generated.stat().st_size == 0:
        raise RuntimeError(f"TensorRT 출력 엔진이 유효하지 않습니다: {generated}")
    shutil.copy2(generated, args.out)
    model_trt = YOLO(str(args.out), task="pose")
    import numpy as np
    warmup = model_trt.predict(np.zeros((args.imgsz, args.imgsz, 3), dtype=np.uint8),
                               imgsz=args.imgsz, device=0, verbose=False)
    if not warmup or warmup[0].keypoints is None:
        raise RuntimeError("TensorRT 엔진의 Pose 출력 검증에 실패했습니다.")
    manifest = {
        "created_at": datetime.now(timezone.utc).isoformat(),
        "architecture": platform.machine(), "gpu_name": torch.cuda.get_device_name(0),
        "jetpack_l4t": Path("/etc/nv_tegra_release").read_text().strip() if Path("/etc/nv_tegra_release").exists() else "check-host",
        "tensorrt_version": trt.__version__, "torch_version": torch.__version__,
        "ultralytics_version": ultralytics.__version__, "precision": "FP16",
        "precision_argument": precision_arg, "imgsz": args.imgsz, "batch": 1,
        "dynamic": False, "pt_sha256": digest(args.pt),
        "day05_onnx_sha256": digest(args.onnx),
        "engine_sha256": digest(args.out), "engine_filename": args.out.name,
        "export_source": "PT model; exporter internally constructs an intermediate ONNX graph",
        "note": "day05 ONNX is checked as a prior artifact, not claimed as the direct engine input",
    }
    (args.out.parent / "engine_manifest.json").write_text(
        json.dumps(manifest, ensure_ascii=False, indent=2), encoding="utf-8")
    print("[PASS] TensorRT engine:", args.out)
    print("[PASS] engine SHA256:", manifest["engine_sha256"])
    print("[PASS] GPU:", manifest["gpu_name"])


if __name__ == "__main__":
    main()
```

**코드 이해하기**

- `YOLO(...).export(format="engine")`는 Ultralytics가 TensorRT 엔진을 생성하도록 요청합니다.
- `quantize=16`은 최근 Ultralytics의 FP16 설정이고, 구버전에서 해당 옵션만 지원하지 않을 때 `half=True`로 동일한 FP16 설정을 시도합니다. **FP32로 자동 대체하지 않습니다.**
- `model_trt.predict()`는 엔진 파일이 **실제로 GPU에서 Pose 출력으로 사용 가능한지** 확인합니다.
- `engine_manifest.json`은 **실제 PT·5일차 ONNX·생성한 엔진의 SHA256**, GPU와 라이브러리 버전을 기록합니다.
- 여기서 5일차 ONNX는 출처를 검증하는 기존 결과물이며, 빌더는 **PT에서 내부 ONNX 중간 그래프를 생성**합니다.

### 실습 8. 같은 이미지의 CUDA·TensorRT 추론시간을 비교하는 코드를 작성합니다

**왜 동일 조건이 필요할까요?** 서로 다른 이미지나 모델 입력 크기를 사용하면 속도 차이의 원인을 알 수 없습니다. 준비한 `test_pose.jpg` 하나를 두 실행 방식에 동일하게 넣고, Warm-up 뒤 반복 측정합니다.

**의사코드**

```text
실제 이미지 한 장 로드
                ↓
같은 YOLO11n-Pose PT를 CUDA로 실행
Warm-up 5회 → 실제 추론 30회 시간 기록
                ↓
같은 모델의 TensorRT 엔진 실행
Warm-up 5회 → 실제 추론 30회 시간 기록
                ↓
p50 · p95 · 평균 지연시간 · 순차 FPS 추정
                ↓
사람 검출 수의 큰 차이가 없는지 확인
                ↓
benchmark.json / benchmark.csv 저장
```

**파일: `deploy/day08_jetson/benchmark_pose.py`**

```python
"""동일 이미지·반복 횟수에서 PyTorch CUDA와 TensorRT의 순차 추론 시간을 비교합니다."""
import argparse
import csv
import json
import statistics
from datetime import datetime, timezone
from pathlib import Path
from time import perf_counter


def run_one(model_file: Path, frame, warmup: int, repeats: int):
    import torch
    from ultralytics import YOLO
    model = YOLO(str(model_file), task="pose")
    for _ in range(warmup):
        model.predict(frame, device=0, imgsz=640, conf=.25, max_det=5, verbose=False)
    torch.cuda.synchronize()
    latencies = []
    counts = []
    for _ in range(repeats):
        torch.cuda.synchronize()
        start = perf_counter()
        result = model.predict(frame, device=0, imgsz=640, conf=.25, max_det=5, verbose=False)[0]
        torch.cuda.synchronize()
        latencies.append((perf_counter() - start) * 1000)
        counts.append(len(result.boxes) if result.boxes is not None else 0)
    ordered = sorted(latencies)
    p95 = ordered[max(0, int((len(ordered) - 1) * .95))]
    del model
    import gc
    gc.collect()
    torch.cuda.empty_cache()
    return {
        "model": model_file.name, "repeats": repeats, "warmup": warmup,
        "mean_ms": round(statistics.mean(latencies), 2),
        "p50_ms": round(statistics.median(latencies), 2),
        "p95_ms": round(p95, 2),
        "serial_fps_estimate": round(repeats / (sum(latencies) / 1000), 2),
        "detections_each_run": counts,
        "latencies_ms": [round(x, 2) for x in latencies],
    }


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--pt", type=Path, required=True)
    parser.add_argument("--engine", type=Path, required=True)
    parser.add_argument("--image", type=Path, required=True)
    parser.add_argument("--out", type=Path, required=True)
    parser.add_argument("--warmup", type=int, default=5)
    parser.add_argument("--repeats", type=int, default=30)
    args = parser.parse_args()
    if args.repeats < 5 or args.warmup < 1:
        raise ValueError("repeats는 5 이상, warmup은 1 이상으로 설정하세요.")
    import cv2
    import torch
    if not torch.cuda.is_available():
        raise RuntimeError("CUDA가 없어 GPU TensorRT 비교를 할 수 없습니다.")
    frame = cv2.imread(str(args.image))
    if frame is None:
        raise FileNotFoundError(f"실제 테스트 이미지를 읽지 못했습니다: {args.image}")
    args.out.mkdir(parents=True, exist_ok=True)
    results = [run_one(f, frame, args.warmup, args.repeats) for f in (args.pt, args.engine)]
    report = {
        "measured_at": datetime.now(timezone.utc).isoformat(),
        "gpu": torch.cuda.get_device_name(0), "image": args.image.name,
        "method": "single-frame sequential YOLO.predict; transfer/storage/display excluded",
        "results": results,
    }
    (args.out / "benchmark.json").write_text(json.dumps(report, indent=2, ensure_ascii=False), encoding="utf-8")
    with (args.out / "benchmark.csv").open("w", encoding="utf-8", newline="") as f:
        writer = csv.writer(f)
        writer.writerow(["model", "mean_ms", "p50_ms", "p95_ms", "serial_fps_estimate", "detections_last"])
        for r in results:
            writer.writerow([r["model"], r["mean_ms"], r["p50_ms"], r["p95_ms"],
                             r["serial_fps_estimate"], r["detections_each_run"][-1]])
    for r in results:
        print(r["model"], "p50_ms", r["p50_ms"], "p95_ms", r["p95_ms"],
              "FPS_est", r["serial_fps_estimate"])
    if results[0]["detections_each_run"][-1] != results[1]["detections_each_run"][-1]:
        print("[주의] 검출 사람 수가 다릅니다. 결과 이미지와 Confidence를 직접 비교하세요.")


if __name__ == "__main__":
    main()
```

`torch.cuda.synchronize()`로 GPU 연산 완료까지 기다린 다음 시간을 계산합니다. `p50`은 지연시간의 중앙값, `p95`는 관찰한 값 중 느린 쪽의 약 95백분위 지연시간을 뜻합니다. `serial_fps_estimate`는 **준비된 동일 이미지의 순차 모델 호출 처리량**입니다. USB/IP 카메라 읽기나 PC로 이미지 전달·DB 저장을 포함하지 않으므로 **실제 카메라 FPS와 구분**합니다.

### 실습 9. Jetson Dockerfile에 9일차 코드를 포함합니다

8일차 **Jetson 전용 Dockerfile**을 수정합니다. 7일차 PC 루트 `Dockerfile`은 변경하지 않습니다. 새 스크립트와 9일차 추론 서버를 컨테이너에 복사하고 시작 파일만 바꿉니다.

**수정 파일: `deploy/day08_jetson/Dockerfile.jetson`**

**파일: `deploy/day08_jetson/Dockerfile.jetson`**

```dockerfile
# JetPack 6 전용 ARM64 CUDA/PyTorch 기반 이미지. PC의 7일차 Dockerfile과 별개입니다.
FROM ultralytics/ultralytics:latest-jetson-jetpack6
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /jetson_app
COPY requirements.jetson.txt ./requirements.jetson.txt
RUN python3 -m pip install --no-cache-dir -r requirements.jetson.txt
COPY app_jetson.py ./app_jetson.py
COPY day09_utils.py ./day09_utils.py
COPY export_tensorrt.py benchmark_pose.py app_day09.py ./
EXPOSE 8018
CMD ["python3", "-m", "uvicorn", "app_day09:app", "--host", "0.0.0.0", "--port", "8018", "--workers", "1"]
```

8일차 `app_jetson.py`는 **삭제하지 않습니다.** `app_day09.py`를 정상 실행하지 못할 때 8일차 이미지·코드로 되돌릴 수 있도록 보관합니다. 이 Dockerfile은 8일차에 사용한 JetPack 6용 GPU 이미지와 패키지를 그대로 상속합니다.

### 실습 10. TensorRT 엔진과 실제 카메라 설정을 Docker Compose에 추가합니다

**수정 파일: `deploy/day08_jetson/compose.jetson.yaml`**

기존 Pose 서비스의 컨테이너 이름·포트 **8018**·GPU Runtime을 그대로 유지합니다. **`/day09/jetson`은 엔진을 쓰고 읽는 볼륨**이며, `/samples`에는 벤치마크 사진을 읽기 전용으로 연결합니다.

**파일: `deploy/day08_jetson/compose.jetson.yaml`**

```yaml
# Jetson 장치에서만 실행. PC 7일차 compose.yaml은 수정하지 않고 별도 파일을 사용합니다.
services:
  pose:
    image: subject13-jetson-pose:day09
    build:
      context: .
      dockerfile: Dockerfile.jetson
    runtime: nvidia
    restart: unless-stopped
    environment:
      JETSON_MODEL_PATH: /models/yolo11n-pose.pt
      DAY09_ENGINE_PATH: /day09/jetson/yolo11n-pose.engine
      DAY09_CAMERA_SOURCE: ${DAY09_CAMERA_SOURCE:-}
      DAY09_TARGET_FPS: ${DAY09_TARGET_FPS:-8}
      JETSON_MODEL_SHA256: ${JETSON_MODEL_SHA256:?모델 파일의 실제 SHA256을 입력하세요}
      JETSON_API_KEY: ${JETSON_API_KEY:?24자 이상의 장치 인증 키를 입력하세요}
    ports:
      - "${JETSON_BIND_HOST:?Jetson LAN IP를 입력하세요}:8018:8018"
    volumes:
      - ../../artifacts/day05/jetson:/models:ro
      - ../../artifacts/day09/jetson:/day09/jetson
      - ./samples:/samples:ro
```

`DAY09_CAMERA_SOURCE`는 카메라 IP URL 또는 USB 번호입니다. **동시에 여러 Pose 컨테이너를 띄우지 않습니다.** 기존 8일차 컨테이너를 동일한 `pose` 서비스로 교체하므로 Jetson 메모리 사용량이 불필요하게 늘어나지 않습니다.

#### USB 카메라를 사용할 때만 추가할 설정

IP Webcam을 사용한다면 아래 추가 파일은 필요하지 않습니다. Jetson에 연결한 USB 카메라가 `/dev/video0`이고 접근이 허용된 경우에만 `deploy/day08_jetson/compose.usb.yaml` 파일을 만듭니다.

```yaml
services:
  pose:
    devices:
      - /dev/video0:/dev/video0
```

USB 실행 때는 기존 Compose 명령에 `-f deploy/day08_jetson/compose.usb.yaml`을 **추가**합니다. 장치의 실제 번호를 `ls -l /dev/video*`로 확인하고 수정합니다. **USB 장치 접근 권한은 Jetson 장치 정책을 따르며 `privileged: true`를 기본값으로 사용하지 않습니다.**

### 실습 11. 9일차 카메라 설정 항목을 추가합니다

**수정 파일: `deploy/day08_jetson/.env.jetson.example`**

예시 파일의 기존 `JETSON_BIND_HOST`, `JETSON_API_KEY`, `JETSON_MODEL_SHA256` 아래에 9일차 설정을 추가합니다. 예시 주소를 그대로 사용할 수는 없습니다.

**파일: `deploy/day08_jetson/.env.jetson.example`**

```dotenv
# 아래 IP/KEY/HASH는 반드시 본인의 Jetson 장치에 맞게 변경합니다.
JETSON_BIND_HOST=192.168.0.31
JETSON_API_KEY=REPLACE_WITH_A_RANDOM_SECRET_AT_LEAST_24_CHARACTERS
JETSON_MODEL_SHA256=REPLACE_WITH_SHA256_OF_YOLO11N_POSE_PT

# 9일차 - IP Webcam 주소 또는 USB 카메라 숫자 0
DAY09_CAMERA_SOURCE=http://192.168.0.50:8080/video
DAY09_TARGET_FPS=8
```

**실제 `.env.jetson`은 Jetson 장치에서만 작성합니다.** 기존 장치 인증 키를 새로 만드는 것이 아니라 8일차 키를 유지합니다. 카메라 주소만 본인의 실제 IP Webcam으로 변경합니다. USB 카메라라면 `DAY09_CAMERA_SOURCE=0`과 USB override Compose 파일을 사용합니다. 인증 키가 들어 있는 `.env.jetson`은 Git에 올리지 않습니다.

---

## PART 3. TensorRT Pose 연속 카메라 서버를 작성합니다

### 실습 12. Jetson TensorRT와 카메라 연속 추론 API를 작성합니다

오늘 새로 작성할 서버는 8일차 API와 **같은 포트 8018**에서 실행됩니다. 아래처럼 정지 이미지 요청도 계속 처리하고, **카메라 최신 프레임 조회 API만 추가**합니다.

```text
GET  /health
  → TensorRT GPU 모델 준비 상태와 카메라 상태

POST /api/v1/jetson/pose
  → 기존 이미지 1장 업로드 경로 유지 (결과 runtime=tensorrt)

GET  /api/v1/jetson/stream/latest
  → 가장 최근에 완료된 카메라 프레임 JPEG·Pose JSON·FPS
```

**의사코드**

```text
[서버 시작]
엔진 파일·Manifest SHA256 검사
TensorRT Pose Warm-up 통과
                ↓
카메라 처리 Thread 시작
                ↓
카메라 열기 → Frame 읽기
                ↓
최대 너비 960px로 축소 → TensorRT Pose 추론
                ↓
BBox / COCO17 / Confidence / 추론시간 산출
                ↓
Frame JPEG와 Pose JSON을 최신 상태로 교체
                ↓
처리 완료 기준 FPS 계산
                ↓
카메라 연결이 끊기면 이전 최신 프레임 제거
2초 뒤 재연결 시도
```

카메라가 끊긴 상태에도 **정지 이미지용 TensorRT API**는 계속 사용할 수 있습니다. 다만 `stream/latest`는 `ready: false`를 반환하여 영상 연결 문제를 숨기지 않습니다. **최근 프레임은 메모리에서만 관리**하며, DB에는 저장 버튼으로 선택한 추론 메타데이터만 기록합니다.

**파일: `deploy/day08_jetson/app_day09.py`**

```python
"""9일차 Jetson: TensorRT Pose 단일 이미지 + 연속 카메라 서비스.

8일차의 HTTP /health 및 /api/v1/jetson/pose 형식은 유지합니다.
"""
import asyncio
import base64
import hashlib
import os
import secrets
import threading
from collections import deque
from contextlib import asynccontextmanager
from datetime import datetime, timezone
from pathlib import Path
from time import monotonic, perf_counter, sleep
from uuid import uuid4

import cv2
import numpy as np
import torch
from fastapi import FastAPI, File, Header, HTTPException, UploadFile
from ultralytics import YOLO

from day09_utils import FrameRate, parse_camera_source

ENGINE = Path(os.getenv("DAY09_ENGINE_PATH", "/day09/jetson/yolo11n-pose.engine"))
MAX_IMAGE_BYTES = 5 * 1024 * 1024
MODEL = None
MODEL_LOCK = threading.Lock()
GPU_NAME = ""
ENGINE_SHA = ""
STOP = threading.Event()
STATE_LOCK = threading.Lock()
STATE = {"camera_status": "starting", "last_error": None, "seq": 0,
         "latest": None, "measured_fps": 0.0, "frames_total": 0}
THREAD = None


def sha256_file(path: Path) -> str:
    h = hashlib.sha256()
    with path.open("rb") as fp:
        for chunk in iter(lambda: fp.read(1048576), b""):
            h.update(chunk)
    return h.hexdigest()


def check_key(key: str | None):
    expected = os.getenv("JETSON_API_KEY", "")
    if len(expected) < 24:
        raise HTTPException(503, "Jetson 장치 API 인증 설정이 없습니다.")
    if not key or not secrets.compare_digest(key, expected):
        raise HTTPException(401, "Jetson 요청 인증에 실패했습니다.")


def load_engine():
    global MODEL, GPU_NAME, ENGINE_SHA
    if not ENGINE.is_file():
        raise RuntimeError(f"9일차 TensorRT 엔진이 없습니다: {ENGINE}")
    manifest_path = ENGINE.parent / "engine_manifest.json"
    if not manifest_path.is_file():
        raise RuntimeError("엔진 manifest가 없습니다. Jetson에서 export_tensorrt.py를 실행하세요.")
    import json
    manifest = json.loads(manifest_path.read_text(encoding="utf-8"))
    actual = sha256_file(ENGINE)
    if actual != manifest.get("engine_sha256") or manifest.get("precision") != "FP16":
        raise RuntimeError("엔진 SHA256 또는 FP16 설정이 일치하지 않습니다.")
    if not torch.cuda.is_available():
        raise RuntimeError("CUDA GPU를 사용할 수 없습니다. CPU로 전환하지 않습니다.")
    candidate = YOLO(str(ENGINE), task="pose")
    result = candidate.predict(np.zeros((640, 640, 3), dtype=np.uint8),
                               device=0, imgsz=640, conf=.25, max_det=5, verbose=False)
    if not result or result[0].keypoints is None:
        raise RuntimeError("TensorRT Pose 17-Keypoint 출력을 확인할 수 없습니다.")
    GPU_NAME = torch.cuda.get_device_name(0)
    ENGINE_SHA = actual
    MODEL = candidate


def infer_frame(frame: np.ndarray, input_ref: str):
    if MODEL is None:
        raise RuntimeError("TensorRT 모델이 준비되지 않았습니다.")
    with MODEL_LOCK:
        start = perf_counter()
        result = MODEL.predict(frame, device=0, imgsz=640, conf=.25,
                               max_det=5, verbose=False)[0]
        # CUDA 작업의 비동기 종료를 포함한 실측 모델 호출 시간
        torch.cuda.synchronize()
        elapsed_ms = round((perf_counter() - start) * 1000, 2)
    persons = []
    boxes = result.boxes
    if boxes is not None and len(boxes) > 0:
        xyxy = boxes.xyxy.cpu().tolist()
        scores = boxes.conf.cpu().tolist()
        kp_xy = result.keypoints.xy.cpu().tolist() if result.keypoints is not None else []
        kp_cf = (result.keypoints.conf.cpu().tolist()
                 if result.keypoints is not None and result.keypoints.conf is not None else [])
        for index, (bbox, score) in enumerate(zip(xyxy, scores)):
            kp = []
            if index < len(kp_xy):
                for joint_index, point in enumerate(kp_xy[index]):
                    conf = kp_cf[index][joint_index] if index < len(kp_cf) else None
                    kp.append([round(float(point[0]), 2), round(float(point[1]), 2),
                               round(float(conf), 4) if conf is not None else None])
            persons.append({"bbox_xyxy": [round(float(x), 2) for x in bbox],
                            "bbox_confidence": round(float(score), 4), "keypoints": kp})
    return {"event_id": str(uuid4()), "recorded_at": datetime.now(timezone.utc).isoformat(),
            "input_ref": Path(input_ref).name, "width": int(frame.shape[1]),
            "height": int(frame.shape[0]), "num_persons": len(persons), "persons": persons,
            "inference_ms": elapsed_ms, "device_id": "jetson-orin-nano-super",
            "model_id": "yolo11n-pose", "model_sha256": ENGINE_SHA,
            "runtime": "tensorrt", "gpu_name": GPU_NAME,
            "note": "TensorRT Pose detection only; action/risk classification is not performed"}


def set_camera_state(status: str, error: str | None = None, clear_latest: bool = False):
    with STATE_LOCK:
        STATE["camera_status"] = status
        STATE["last_error"] = error
        if clear_latest:
            STATE["latest"] = None
            STATE["measured_fps"] = 0.0


def camera_loop():
    source_text = os.getenv("DAY09_CAMERA_SOURCE", "").strip()
    try:
        source = parse_camera_source(source_text)
    except ValueError as exc:
        set_camera_state("configuration_error", str(exc), True)
        return
    target_fps = max(1.0, min(15.0, float(os.getenv("DAY09_TARGET_FPS", "8"))))
    fps = FrameRate()
    while not STOP.is_set():
        set_camera_state("connecting", clear_latest=True)
        stream = cv2.VideoCapture(source)
        if not stream.isOpened():
            stream.release()
            set_camera_state("reconnecting", "카메라 연결 실패", True)
            STOP.wait(2.0)
            continue
        stream.set(cv2.CAP_PROP_BUFFERSIZE, 1)
        set_camera_state("running")
        bad_reads = 0
        while not STOP.is_set():
            started = monotonic()
            ok, frame = stream.read()
            if not ok or frame is None:
                bad_reads += 1
                if bad_reads >= 3:
                    set_camera_state("reconnecting", "프레임 읽기 실패", True)
                    break
                STOP.wait(.15)
                continue
            bad_reads = 0
            height, width = frame.shape[:2]
            if width > 960:
                new_height = max(1, round(height * 960 / width))
                frame = cv2.resize(frame, (960, new_height))
            try:
                pose = infer_frame(frame, "day09_camera_frame.jpg")
                success, jpeg = cv2.imencode(".jpg", frame, [cv2.IMWRITE_JPEG_QUALITY, 70])
                if not success:
                    raise RuntimeError("카메라 JPEG 변환 실패")
                encoded = base64.b64encode(jpeg.tobytes()).decode("ascii")
                with STATE_LOCK:
                    STATE["seq"] += 1
                    STATE["frames_total"] += 1
                    STATE["measured_fps"] = fps.tick()
                    STATE["camera_status"] = "running"
                    STATE["last_error"] = None
                    STATE["latest"] = {"pose": pose, "jpeg_base64": encoded,
                                       "frame_seq": STATE["seq"], "created_monotonic": monotonic()}
            except Exception as exc:
                # 새 프레임이 없다면 오래된 이미지가 최신 화면으로 보이지 않게 합니다.
                set_camera_state("processing_error", type(exc).__name__, True)
                break
            STOP.wait(max(0.0, 1.0 / target_fps - (monotonic() - started)))
        stream.release()
        STOP.wait(2.0)


@asynccontextmanager
async def lifespan(app: FastAPI):
    global THREAD
    load_engine()
    STOP.clear()
    THREAD = threading.Thread(target=camera_loop, name="jetson-pose-camera", daemon=True)
    THREAD.start()
    yield
    STOP.set()
    if THREAD is not None:
        THREAD.join(timeout=3)


app = FastAPI(title="교과13 Jetson TensorRT Pose", version="day09", lifespan=lifespan)


@app.get("/health")
def health():
    with STATE_LOCK:
        status = STATE["camera_status"]
    return {"status": "ready" if MODEL is not None else "not_ready",
            "gpu_available": torch.cuda.is_available(), "gpu_name": GPU_NAME,
            "model_id": "yolo11n-pose", "runtime": "tensorrt",
            "device": "jetson", "camera_status": status}


@app.post("/api/v1/jetson/pose")
async def pose_image(image: UploadFile = File(...),
                     x_jetson_key: str | None = Header(default=None, alias="X-Jetson-Key")):
    check_key(x_jetson_key)
    if image.content_type not in {"image/jpeg", "image/png"}:
        raise HTTPException(415, "JPG/PNG만 받습니다.")
    raw = await image.read(MAX_IMAGE_BYTES + 1)
    if not raw or len(raw) > MAX_IMAGE_BYTES:
        raise HTTPException(413, "비어 있거나 5MB를 초과하는 이미지입니다.")
    frame = cv2.imdecode(np.frombuffer(raw, dtype=np.uint8), cv2.IMREAD_COLOR)
    if frame is None:
        raise HTTPException(422, "이미지를 읽을 수 없습니다.")
    if frame.shape[0] * frame.shape[1] > 4096 * 4096:
        raise HTTPException(413, "이미지 해상도가 너무 큽니다.")
    return await asyncio.to_thread(infer_frame, frame, Path(image.filename or "pose.jpg").name)


@app.get("/api/v1/jetson/stream/latest")
def latest(x_jetson_key: str | None = Header(default=None, alias="X-Jetson-Key")):
    check_key(x_jetson_key)
    with STATE_LOCK:
        saved = STATE["latest"]
        status = STATE["camera_status"]
        fps = STATE["measured_fps"]
        count = STATE["frames_total"]
        error = STATE["last_error"]
    if saved is None or monotonic() - saved["created_monotonic"] > 3.0:
        return {"ready": False, "camera_status": status, "measured_fps": fps,
                "frames_total": count, "last_error": error, "pose": None,
                "jpeg_base64": None, "frame_seq": None}
    return {"ready": True, "camera_status": status, "measured_fps": fps,
            "frames_total": count, "last_error": error,
            "pose": saved["pose"], "jpeg_base64": saved["jpeg_base64"],
            "frame_seq": saved["frame_seq"]}
```

**코드 이해하기**

- `load_engine()`은 **FP16 엔진과 SHA256 Manifest가 일치하지 않으면 시작을 중단**합니다.
- `MODEL_LOCK`은 카메라 Thread와 정지 이미지 API가 동시에 한 GPU 모델을 호출하지 않게 보호합니다.
- `infer_frame()`은 8일차와 같은 `PoseResponse` 주요 필드를 반환하지만, `runtime="tensorrt"`와 **엔진 SHA256**을 기록합니다.
- `FrameRate`는 카메라 프레임 분석이 완료된 시각으로 측정합니다.
- `STATE["latest"]`에는 화면 표시용 최신 프레임만 보유합니다. **프레임을 디스크에 무한히 저장하지 않습니다.**
- `STOP.wait(2.0)`은 카메라가 끊겼을 때 재연결 전 잠시 기다립니다.
- 연속 프레임이 **3초 넘게 오래되면** 최신 화면으로 반환하지 않습니다.

**이번 실습의 중요한 구분:** `pose.persons[].bbox_confidence`는 **사람 검출 신뢰도**입니다. 6개 Action 확률이나 작업자의 위험도 점수가 아닙니다. 사람을 발견했다고 `OK`, 찾지 못했다고 `NG`로 자동 판단하지 않습니다.

---

## PART 4. Jetson 장치에서 엔진·벤치마크·카메라를 실행합니다

### 실습 13. PC에서 수정한 Jetson 소스 파일을 전송합니다

8일차에 Jetson으로 `rsync`했던 동일한 폴더를 사용합니다. 먼저 **PC 터미널**에서 실제 장치 계정·IP를 설정합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
export JETSON_USER=student
export JETSON_IP=실제_Jetson_LAN_IP

ssh "${JETSON_USER}@${JETSON_IP}" \
  'mkdir -p ~/ai_vision/subject13_ai_platform/artifacts/day09/jetson'

rsync -av --exclude='__pycache__/' --exclude='*.pyc' \
  deploy/day08_jetson/ \
  "${JETSON_USER}@${JETSON_IP}:~/ai_vision/subject13_ai_platform/deploy/day08_jetson/"
```

`JETSON_USER`, `JETSON_IP`는 예시이며 실제 장치에 맞게 바꿉니다. 8일차에 사용한 `.pt`, `.onnx`, `test_pose.jpg`는 이미 Jetson에 있는지 다음 실습에서 확인합니다. 누락되었다면 8일차의 **해당 파일 복사 단계만** 다시 실행합니다.

### 실습 14. Jetson의 모델·테스트 이미지와 쓰기 가능한 폴더를 확인합니다

**Jetson 터미널**에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
mkdir -p artifacts/day09/jetson artifacts/day09/benchmarks
ls -lh artifacts/day05/jetson/yolo11n-pose.pt \
  artifacts/day05/jetson/yolo11n-pose.onnx \
  deploy/day08_jetson/samples/test_pose.jpg
```

**엔진은 이 Jetson 장치에서 새로 만듭니다.** 기존 `.pt`와 `.onnx` 파일의 SHA256이 5일차 결과와 같은지도 확인합니다.

```bash
sha256sum artifacts/day05/jetson/yolo11n-pose.pt \
  artifacts/day05/jetson/yolo11n-pose.onnx
```

### 실습 15. Compose 문법 확인 후 9일차 GPU 이미지를 빌드합니다

Jetson의 `.env.jetson`을 아직 변경하지 않아도 다음 명령으로 이미지 구성을 먼저 검사하고 빌드할 수 있습니다. 단, `.env.jetson`에 8일차의 실제 인증 키·모델 해시가 이미 설정되어 있어야 합니다.

```bash
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml config --quiet

docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml build pose
```

이미지를 새로 빌드해도 **현재 실행 중인 8일차 컨테이너가 즉시 교체되지는 않습니다.** 오늘은 새 이미지를 `subject13-jetson-pose:day09`로 만들고 **이전 `:day08` 이미지는 보존**합니다. TensorRT 엔진을 준비한 다음 서비스를 전환합니다.

### 실습 15-1. 엔진 생성 전에 기존 Jetson Pose를 중지합니다

엔진 생성은 GPU 메모리를 많이 사용할 수 있습니다. **Jetson의 Pose 서비스만 중지**하여 GPU 메모리를 확보하고, PC의 DB·Redis·FastAPI는 그대로 둡니다. 이때 사진 추론은 잠시 중단되는 것이 정상입니다.

```bash
# Jetson 터미널
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml stop pose

docker images | grep subject13-jetson-pose
```

이미지 목록에서 **`subject13-jetson-pose:day08`과 `:day09` 두 이미지가 존재**하는지 확인합니다. `:day08`을 삭제하지 않습니다. TensorRT 빌드에 실패하더라도 8일차 이미지로 복구할 수 있어야 합니다.

### 실습 16. Jetson에서 실제 TensorRT FP16 엔진을 생성합니다

이제 빌드한 이미지에서 일회성 컨테이너를 실행합니다. 이 컨테이너는 8일차 Pose 서버를 덮어쓰지 않고 **엔진 생성 스크립트만 실행**합니다.

```bash
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml   run --rm --no-deps --entrypoint python3 pose   export_tensorrt.py   --pt /models/yolo11n-pose.pt   --onnx /models/yolo11n-pose.onnx   --out /day09/jetson/yolo11n-pose.engine   --imgsz 640
```

처음 생성할 때 GPU 메모리와 TensorRT 엔진 빌드 시간이 필요할 수 있습니다. `No module named tensorrt`, CUDA 비활성화, 엔진 출력 누락 등이 발생했다면 오류를 해결한 뒤 다시 실행합니다. **성공했다고 가정하고 다음으로 넘어가지 않습니다.**

**Jetson 터미널**에서 두 파일이 실제 생성되었는지 확인합니다.

```bash
ls -lh artifacts/day09/jetson/yolo11n-pose.engine   artifacts/day09/jetson/engine_manifest.json
sha256sum artifacts/day09/jetson/yolo11n-pose.engine
python3 -m json.tool artifacts/day09/jetson/engine_manifest.json | head -35
```

정상 결과에는 실제 GPU 이름, TensorRT 버전, FP16 여부, 엔진 SHA256이 포함되어야 합니다. 엔진 파일이 없거나 크기가 0이면 실습이 완료된 것이 아닙니다.

### 실습 17. 동일 이미지로 GPU 두 실행 방식의 성능을 비교합니다

실습 15-1에서 기존 8일차 Jetson GPU 서비스를 중지했으므로, 일회성 컨테이너로 **같은 사람 이미지 한 장**을 반복 추론합니다. 워밍업 5회와 본 측정 30회로 구분합니다.

```bash
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml   run --rm --no-deps --entrypoint python3 pose   benchmark_pose.py   --pt /models/yolo11n-pose.pt   --engine /day09/jetson/yolo11n-pose.engine   --image /samples/test_pose.jpg   --out /day09/jetson/benchmarks   --warmup 5 --repeats 30
```

**Jetson 터미널**에서 결과 파일을 확인합니다.

```bash
ls -lh artifacts/day09/jetson/benchmarks/
cat artifacts/day09/jetson/benchmarks/benchmark.csv
```

콘솔과 CSV에는 **직접 실행한 Jetson의 숫자**가 저장됩니다. **교재는 임의의 p50·p95·FPS 값을 제공하지 않습니다.** GPU 성능을 비교할 때 다른 학습·추론 작업이 실행 중이면 결과가 흔들릴 수 있습니다. **8일차 서버는 실습 15-1에서 중지한 상태**여야 하고, PC의 DB 및 Redis 서비스는 중단하지 않습니다.

**결과를 해석합니다.**

| 비교 항목 | PyTorch CUDA (`.pt`) | TensorRT (`.engine`) |
|---|---|---|
| 모델 | 동일 `yolo11n-pose` | 동일 `yolo11n-pose` |
| 시험 입력 | `test_pose.jpg` | 동일 이미지 |
| 입력 설정 | `imgsz=640`, `conf=0.25` | 동일 설정 |
| p50 / p95(ms) | 실제 CSV 값 기록 | 실제 CSV 값 기록 |
| 순차 FPS 추정 | 실제 CSV 값 기록 | 실제 CSV 값 기록 |
| 검출 사람 수 | 실제 `detections_each_run` | 동일 이미지 결과 비교 |

이 표의 **실제 숫자는 학생의 Jetson 측정 결과로만 채웁니다.** FP16 때문에 모델의 일부 출력이 달라질 수 있습니다. 사람 수가 다르게 나왔다면 결과가 무조건 틀리거나 동일하다고 단정하지 말고 두 실행 결과를 직접 확인합니다.

### 실습 18. Jetson `.env.jetson`에 실제 카메라 주소를 설정합니다

**Jetson 터미널**에서 원래 8일차에 만든 `.env.jetson`을 엽니다.

```bash
nano .env.jetson
```

기존 `JETSON_BIND_HOST`, `JETSON_API_KEY`, `JETSON_MODEL_SHA256`은 유지하면서 아래 두 항목을 추가합니다. IP 주소는 **본인의 IP Webcam 실제 주소**로 바꿉니다.

```dotenv
DAY09_CAMERA_SOURCE=http://실제_IPWebcam_IP:8080/video
DAY09_TARGET_FPS=8
```

이 설정은 Jetson이 IP Webcam 주소에 직접 접속한다는 뜻입니다. 사용자의 PC 또는 휴대전화에서만 접근되는 주소가 아니라 **Jetson에서 접근 가능한 주소**여야 합니다.

USB 카메라를 사용하는 학생은 아래처럼 입력합니다.

```dotenv
DAY09_CAMERA_SOURCE=0
DAY09_TARGET_FPS=8
```

USB를 사용할 때는 **실습 10의 `compose.usb.yaml`과 장치 매핑**을 함께 적용해야 합니다. 숫자 `0`이 항상 연결된 카메라는 아니므로 Jetson에서 `ls -l /dev/video*`로 실제 장치를 확인합니다.

### 실습 19. Jetson의 Pose 컨테이너를 9일차 TensorRT 서버로 교체합니다

TensorRT 엔진과 카메라 설정까지 준비했다면 기존 `pose` 서비스를 **동일한 서비스 이름·동일한 포트로 교체**합니다. 다른 이름의 GPU 추론 서버를 별도로 띄우지 않습니다.

**IP Webcam 사용 시:**

```bash
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml   up -d --no-deps pose

docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml ps
```

**USB 카메라 사용 시:** 위 두 명령에서 `-f deploy/day08_jetson/compose.jetson.yaml` 뒤에 `-f deploy/day08_jetson/compose.usb.yaml`을 각각 추가합니다.

시작 로그를 확인합니다.

```bash
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml   logs --tail=80 pose
```

**정상 기준:** TensorRT 엔진 SHA256 확인·GPU Warm-up을 통과하고 `pose` 서비스가 실행되어야 합니다. 모델이 잘못되어 시작하지 못하면 오류가 로그에 기록되어야 하며, CPU 모델로 몰래 바뀌지 않습니다.

### 실습 20. TensorRT 서버와 정지 이미지 Pose API를 확인합니다

**Jetson 터미널**에서 본인의 실제 장치 IP를 사용합니다.

```bash
export JETSON_HOST_IP=실제_Jetson_LAN_IP
curl --fail "http://${JETSON_HOST_IP}:8018/health" | python3 -m json.tool
```

정상일 때 `runtime: "tensorrt"`, `gpu_available: true`, `status: "ready"`가 출력됩니다. 카메라 연결이 아직 안 되었다면 `camera_status`가 `connecting` 또는 `reconnecting`일 수 있지만, 이는 **엔진 로딩 실패와 다른 오류**입니다.

8일차와 동일한 이미지 업로드 주소도 계속 동작하는지 확인합니다.

```bash
curl --fail --max-time 65 -X POST   "http://${JETSON_HOST_IP}:8018/api/v1/jetson/pose"   -H "X-Jetson-Key: $(sed -n 's/^JETSON_API_KEY=//p' .env.jetson)"   -F "image=@deploy/day08_jetson/samples/test_pose.jpg;type=image/jpeg"   | python3 -m json.tool
```

응답의 `runtime`이 `tensorrt`이고 `model_sha256`이 엔진 Manifest SHA256과 일치하는지 확인합니다. 좌표·사람 수·추론 시간은 실제 실행 결과를 사용합니다.

### 실습 21. Jetson 연속 카메라 최신 프레임을 확인합니다

**Jetson 터미널**에서 같은 장치 IP를 사용합니다.

```bash
curl --fail --max-time 10   -H "X-Jetson-Key: $(sed -n 's/^JETSON_API_KEY=//p' .env.jetson)"   "http://${JETSON_HOST_IP}:8018/api/v1/jetson/stream/latest"   | python3 -c 'import json,sys; d=json.load(sys.stdin); print("ready",d["ready"],"camera",d["camera_status"],"fps",d["measured_fps"],"frame_seq",d["frame_seq"],"persons",d["pose"]["num_persons"] if d["pose"] else None)'
```

정상적으로 프레임을 읽고 있다면 `ready True`, `camera running`, 연속적으로 증가하는 `frame_seq`가 나타납니다. 몇 초 간격으로 두 번 실행하면 `frame_seq`가 증가하는지 확인합니다.

`ready False`라면 카메라 주소·네트워크·장치 권한을 확인합니다. **기존 테스트 이미지의 결과를 카메라 최신 프레임처럼 대신 표시하지 않습니다.**

---

## PART 5. 8일차 PC FastAPI에 TensorRT·연속 프레임을 연결합니다

### 실습 22. 8일차 공통 이벤트 변환에서 TensorRT를 허용합니다

**PC 터미널**로 이동하여 기존 프로젝트의 `app/jetson_api.py`를 엽니다. 8일차 코드 전체를 삭제하지 않습니다.

**수정 파일: `app/jetson_api.py`**

먼저 `pose_to_event()` 함수의 모델·실행환경 확인 부분을 찾습니다.

**8일차 기존 부분**

```python
if pose.model_id != "yolo11n-pose" or pose.runtime != "pytorch_cuda":
    raise ValueError("8일차의 YOLO11n Pose/CUDA 응답이 아닙니다.")
```

**다음처럼 수정합니다.**

```python
if pose.model_id != "yolo11n-pose" or pose.runtime not in {"pytorch_cuda", "tensorrt"}:
    raise ValueError("YOLO11n Pose CUDA/TensorRT GPU 응답이 아닙니다.")
```

`return InferenceEvent(...)` 바로 앞에 8·9일차 모델 파일 출처와 버전 식별 문자열을 지정합니다.

```python
origin = ("artifacts/day09/jetson/yolo11n-pose.engine"
          if pose.runtime == "tensorrt" else "artifacts/day05/jetson/yolo11n-pose.pt")
version = "day09-trt" if pose.runtime == "tensorrt" else "day08-cuda"
```

그리고 `InferenceEvent(...)`의 아래 **두 항목만** 바꿉니다.

```python
source_project="subject13_ai_platform", source_path=origin,
# 다른 기존 필드는 그대로 둡니다.
processor_id=pose.model_id, processor_version=f"{version}-{pose.model_sha256[:12]}",
```

여기에 나온 두 줄은 **함수 내부에서 기존 항목을 교체하기 위한 부분 코드**입니다. 별도로 중복 삽입하지 않습니다.

이렇게 하면 8일차의 `.pt` GPU 응답과 오늘의 `.engine` TensorRT 응답이 **서로 다른 모델 실행 버전**으로 같은 PostgreSQL에 저장됩니다. `decision=UNKNOWN`, `reasons=POSE_ONLY_NO_ACTION_DECISION` 원칙은 변경하지 않습니다.

### 실습 23. PC의 새 최신 프레임 조회·저장 API를 작성합니다

Jetson이 만든 연속 프레임을 PC FastAPI에서 받아 화면에 전달합니다. 사용자가 `DB 저장` 버튼을 누르면 **그 시점의 가장 최근에 준비된 프레임 하나**를 8일차의 `save_jetson_event()`로 저장합니다. 기존 DB 테이블은 수정하지 않습니다.

**의사코드**

```text
PC가 Jetson /stream/latest 요청
                ↓
공유 키·모델 runtime·Pose JSON·JPEG 형식 검증
                ↓
GET /api/v1/jetson/stream/latest
→ PC 브라우저에 최신 프레임 전달
                ↓
사용자가 저장 버튼 클릭
                ↓
POST /api/v1/jetson/stream/save
                ↓
가장 최근 Pose → 공통 InferenceEvent 변환
                ↓
기존 save_jetson_event() 호출
                ↓
기존 PostgreSQL inference_events 저장
```

`/stream/save`는 **버튼 클릭 시점에 Jetson이 가진 가장 최근 프레임**을 저장합니다. 브라우저가 마지막으로 표시한 이미지와 네트워크 지연 때문에 약간 다를 수 있으므로, 응답의 `event_id`와 `frame_seq`를 확인합니다. **모든 프레임을 자동으로 DB에 저장하지 않습니다.**

**파일: `app/day09_api.py`**

```python
"""9일차: Jetson 연속 카메라 최신 프레임 중계 및 선택적 PostgreSQL 보존."""
import base64
from pathlib import Path

import httpx
from fastapi import APIRouter, Depends, HTTPException
from fastapi.responses import FileResponse
from pydantic import BaseModel, Field, model_validator
from sqlalchemy.orm import Session

from app.db import get_db
from app.jetson_api import PoseResponse, jetson_connection, pose_to_event
from app.jetson_repo import save_jetson_event

router = APIRouter(tags=["day09-tensorrt-stream"])
ROOT = Path(__file__).resolve().parents[1]
MAX_BASE64_CHARS = 2_000_000


class LatestFrame(BaseModel):
    ready: bool
    camera_status: str
    measured_fps: float = Field(ge=0)
    frames_total: int = Field(ge=0)
    last_error: str | None = None
    pose: PoseResponse | None = None
    jpeg_base64: str | None = Field(default=None, max_length=MAX_BASE64_CHARS)
    frame_seq: int | None = None

    @model_validator(mode="after")
    def validate_ready_frame(self):
        if self.ready:
            if self.pose is None or self.jpeg_base64 is None or self.frame_seq is None:
                raise ValueError("준비된 프레임에는 Pose·JPEG·시퀀스가 있어야 합니다.")
            if self.pose.runtime != "tensorrt":
                raise ValueError("TensorRT 프레임이 아닙니다.")
            if len(self.pose.persons) != self.pose.num_persons:
                raise ValueError("사람 수가 일치하지 않습니다.")
            if any(len(p.keypoints) not in (0, 17) for p in self.pose.persons):
                raise ValueError("Pose Keypoint 수가 잘못되었습니다.")
            try:
                raw = base64.b64decode(self.jpeg_base64, validate=True)
            except (ValueError, base64.binascii.Error) as exc:
                raise ValueError("이미지 base64가 잘못되었습니다.") from exc
            if not raw.startswith(b"\xff\xd8\xff"):
                raise ValueError("이미지 형식이 JPEG가 아닙니다.")
        return self


def fetch_latest() -> LatestFrame:
    url, key = jetson_connection()
    try:
        with httpx.Client(timeout=8) as client:
            response = client.get(url + "/api/v1/jetson/stream/latest",
                                  headers={"X-Jetson-Key": key})
            response.raise_for_status()
        return LatestFrame.model_validate(response.json())
    except httpx.RequestError as exc:
        raise HTTPException(503, "Jetson 카메라 서비스에 연결할 수 없습니다.") from exc
    except (httpx.HTTPStatusError, ValueError) as exc:
        raise HTTPException(502, "Jetson 카메라 응답을 검증하지 못했습니다.") from exc


@router.get("/jetson-stream")
def stream_page():
    return FileResponse(ROOT / "web" / "jetson_stream.html")


@router.get("/api/v1/jetson/stream/latest", response_model=LatestFrame)
def stream_latest():
    return fetch_latest()


@router.post("/api/v1/jetson/stream/save")
def stream_save(db: Session = Depends(get_db)):
    """저장 버튼을 누른 순간 Jetson이 가진 가장 최신 프레임 1건을 DB에 기록합니다."""
    item = fetch_latest()
    if not item.ready or item.pose is None:
        raise HTTPException(503, "현재 카메라 프레임이 없습니다. 연결을 확인하세요.")
    event = pose_to_event(item.pose)
    saved, created = save_jetson_event(db, event)
    return {"saved": created, "event_id": saved.event_id,
            "frame_seq": item.frame_seq, "decision": saved.decision,
            "runtime": saved.details.get("runtime")}
```

**코드 이해하기**

- `LatestFrame`은 Jetson의 JSON을 검증합니다. 모델 실행 방식이 `tensorrt`가 아니거나 Keypoint 개수가 잘못되면 받아들이지 않습니다.
- `fetch_latest()`는 **기존 8일차 Jetson IP와 공유 키 설정**을 그대로 사용합니다.
- `stream_save()`는 8일차 `pose_to_event()`와 `save_jetson_event()`를 다시 사용합니다. 중복된 `event_id`를 받으면 새로운 DB 행을 무조건 늘리지 않습니다.
- 이미지의 Base64 바이트는 브라우저 표시용으로만 전달합니다. 기존 DB에는 Pose 메타데이터만 기록합니다.
- 카메라 준비가 되지 않았는데 저장 버튼을 누르면 **HTTP 503**으로 중단합니다.

### 실습 24. 기존 `app/main.py`에 9일차 Router를 추가합니다

**수정 파일: `app/main.py`**

이미 포함된 1~8일차 Router는 **삭제하지 않습니다.** 기존 파일 아래에 다음 두 줄만 추가합니다.

```python
from app.day09_api import router as day09_router
app.include_router(day09_router)
```

한 번만 추가하면 됩니다. 중복으로 두 번 등록하지 않습니다.

### 실습 25. PC에서 Jetson 연결 환경변수를 확인합니다

**PC 프로젝트의 기존 `.env`**에 8일차에서 추가한 다음 값이 이미 있어야 합니다.

```dotenv
DAY08_JETSON_URL=http://실제_Jetson_LAN_IP:8018
DAY08_JETSON_API_KEY=Jetson_dot_env_jetson과_일치하는_실제_키
```

새 포트나 새로운 보안 키를 만드는 것이 아닙니다. 오늘 Jetson 서비스는 **같은 LAN IP·포트 8018**을 사용합니다. PC 루트의 `compose.yaml`에도 8일차에 추가한 아래 두 변수가 `services → api → environment`에 있는지 확인합니다.

```yaml
DAY08_JETSON_URL: ${DAY08_JETSON_URL:-}
DAY08_JETSON_API_KEY: ${DAY08_JETSON_API_KEY:-}
```

1~8일차 `DB_HOST`, `REDIS_URL`, Raspberry Pi 설정 등은 그대로 유지합니다. `.env`의 인증 키를 README에 기록하거나 Git에 커밋하지 않습니다.

---

## PART 6. PC 브라우저에서 최신 프레임·Skeleton과 실측 FPS를 확인합니다

### 실습 26. 연속 카메라 웹 화면을 작성합니다

오늘 화면에서는 Jetson의 연속 Pose 결과를 주기적으로 읽고 Canvas에 Skeleton을 표시합니다. **8일차에 만든 `web/jetson.html`은 그대로 두고**, 새 페이지 하나를 추가합니다.

**의사코드**

```text
/jetson-stream 브라우저 열기
                ↓
0.5초마다 PC /stream/latest 조회
                ↓
JPEG를 Canvas에 그림
                ↓
Pose bbox_xyxy / 17 Keypoint 선·점 표시
                ↓
처리 FPS / 추론 ms / 사람 수 표시
                ↓
[최근 프레임 DB 저장] 클릭
                ↓
PC FastAPI → PostgreSQL 저장 → event_id 표시
                ↓
카메라 끊김 → 오류·재연결 상태 확인
```

**파일: `web/jetson_stream.html`**

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>교과 13 · Jetson TensorRT 연속 Pose</title>
  <style>
    body {font-family:system-ui,sans-serif;max-width:1020px;margin:30px auto;padding:0 15px;color:#233044}
    .panel {border:1px solid #cbd5e1;border-radius:10px;padding:16px;margin:12px 0}
    canvas {width:100%;max-width:960px;background:#111827;display:block}
    button {padding:9px 15px;margin:8px 6px 8px 0;cursor:pointer}
    pre {white-space:pre-wrap;overflow-wrap:anywhere;background:#f1f5f9;padding:12px}
    .warn {color:#ad3d20}
  </style>
</head>
<body>
  <h1>교과 13 · 9일차 TensorRT Pose</h1>
  <p>Jetson이 카메라 프레임을 연속 분석합니다. 브라우저는 0.5초마다 최신 프레임을 가져옵니다.</p>
  <p class="warn">표시된 Person Pose는 동작인식 6종 또는 산업안전 OK/NG 판정이 아닙니다.</p>
  <div class="panel">
    <button id="toggle">화면 갱신 일시중지</button>
    <button id="save">최근 프레임 DB 저장</button>
    <strong id="status">연결 확인 중...</strong>
    <p id="stats">—</p>
    <canvas id="canvas" width="960" height="540"></canvas>
  </div>
  <div class="panel">
    <h2>현재 프레임 정보</h2>
    <pre id="info">연결 중</pre>
    <h2>DB 저장 결과</h2>
    <pre id="saved">아직 저장하지 않았습니다.</pre>
  </div>
<script>
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const status = document.getElementById('status');
const stats = document.getElementById('stats');
const info = document.getElementById('info');
const saved = document.getElementById('saved');
let running = true;
let busy = false;
let lastDrawn = -1;
const edges = [[5,7],[7,9],[6,8],[8,10],[5,6],[5,11],[6,12],
               [11,12],[11,13],[13,15],[12,14],[14,16],[0,1],[0,2],[1,3],[2,4]];
function paintPose(persons) {
  for (const person of persons) {
    const box=person.bbox_xyxy;
    ctx.strokeStyle='#29f09b';ctx.lineWidth=2;
    ctx.strokeRect(box[0],box[1],box[2]-box[0],box[3]-box[1]);
    const pts=person.keypoints || [];
    function valid(index) {
      if (!pts[index]) return false;
      const p=pts[index];
      return Number.isFinite(p[0]) && Number.isFinite(p[1]) &&
             (p[2] === null || p[2] >= 0.3) && !(p[0] === 0 && p[1] === 0);
    }
    ctx.strokeStyle='#fbcc40';ctx.fillStyle='#fbcc40';ctx.lineWidth=2;
    for (const [a,b] of edges) {
      if (!valid(a)||!valid(b)) continue;
      ctx.beginPath();ctx.moveTo(pts[a][0],pts[a][1]);ctx.lineTo(pts[b][0],pts[b][1]);ctx.stroke();
    }
    pts.forEach((p,i)=>{if(!valid(i))return;ctx.beginPath();ctx.arc(p[0],p[1],3,0,Math.PI*2);ctx.fill();});
  }
}
async function update() {
  if (!running || busy) return;
  busy = true;
  try {
    const response=await fetch('/api/v1/jetson/stream/latest',{cache:'no-store'});
    if(!response.ok)throw Error('PC API HTTP '+response.status);
    const obj=await response.json();
    if(!obj.ready){
      status.textContent='카메라: '+obj.camera_status;
      stats.textContent=obj.last_error || '재연결 대기 중';
      ctx.clearRect(0,0,canvas.width,canvas.height);
      info.textContent='카메라 연결이 준비되지 않았습니다. 이전 프레임은 표시하지 않습니다.';
      return;
    }
    const pose=obj.pose;
    const image=new Image();
    image.onload=()=>{
      if(obj.frame_seq < lastDrawn)return;
      lastDrawn=obj.frame_seq;
      canvas.width=pose.width;canvas.height=pose.height;
      ctx.drawImage(image,0,0,pose.width,pose.height);
      paintPose(pose.persons);
    };
    image.src='data:image/jpeg;base64,'+obj.jpeg_base64;
    status.textContent='연속 분석 중 · TensorRT';
    stats.textContent=`순번 ${obj.frame_seq} · 처리 FPS ${obj.measured_fps} · 추론 ${pose.inference_ms} ms · 사람 ${pose.num_persons}명`;
    info.textContent=JSON.stringify({event_id:pose.event_id,recorded_at:pose.recorded_at,
       runtime:pose.runtime,model_sha256:pose.model_sha256,gpu:pose.gpu_name,
       num_persons:pose.num_persons,decision:'UNKNOWN (Pose만 수행)'},null,2);
  } catch(error){
    status.textContent='연결 오류';stats.textContent=error.message;
    ctx.clearRect(0,0,canvas.width,canvas.height);
  }
  finally {busy=false;}
}
document.getElementById('toggle').addEventListener('click',event=>{
  running=!running;event.target.textContent=running?'화면 갱신 일시중지':'화면 갱신 재개';
  if(running)update();
});
document.getElementById('save').addEventListener('click',async()=>{
  try{
    const response=await fetch('/api/v1/jetson/stream/save',{method:'POST'});
    const data=await response.json();
    if(!response.ok)throw Error(data.detail || 'DB 저장 실패');
    saved.textContent=JSON.stringify(data,null,2);
  }catch(error){saved.textContent=error.message;}
});
setInterval(update,500);
update();
</script>
</body>
</html>
```

**웹 코드 이해하기**

`fetch()`로 최신 Pose를 요청하고, `jpeg_base64`를 브라우저의 JPEG 이미지로 표시합니다. 좌표는 **해당 프레임의 픽셀 위치**이므로 Canvas의 실제 내부 `width`, `height`를 `pose.width`, `pose.height`로 맞춥니다. 각 Keypoint의 신뢰도가 낮으면 연결선을 그리지 않으며, 신뢰도가 없는 값(`null`)은 추측해 만들어 내지 않습니다.

`setInterval(update, 500)`은 **웹 갱신 주기**입니다. Jetson 모델의 `measured_fps`와 같다는 뜻이 아닙니다. **카메라 연결 상태가 `ready: false`면** Canvas의 이전 프레임을 지우고 재연결 상태를 보여줍니다. 이미지 응답이 늦게 도착해도 `frame_seq`를 비교하여 더 오래된 프레임이 최근 프레임을 덮어쓰지 않도록 합니다.

### 실습 27. PC FastAPI 이미지를 재빌드하고 정상 실행을 확인합니다

새 Python Router·웹 페이지를 Docker 이미지에 반영해야 합니다. **PC 터미널**에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
docker compose config --quiet
docker compose up -d --build api
docker compose ps
```

DB·Redis·Celery Worker를 삭제하지 않습니다. 이후 PC 웹브라우저에서 다음 주소를 엽니다.

```text
http://127.0.0.1:8000/jetson-stream
```

모델·카메라가 정상이라면 **새 프레임이 나타나고 Bounding Box·Skeleton이 갱신**되어야 합니다. 브라우저에 다음 네 정보를 직접 확인합니다.

- `runtime: tensorrt` — 8일차 PyTorch CUDA가 아닌 실제 TensorRT 실행기인지
- `measured_fps` — Jetson 연속 프레임 처리의 실제 값인지
- `inference_ms` — 지금 분석한 프레임의 모델 호출 시간인지
- `num_persons` — 화면에 나타난 사람의 검출 수인지

사람이 나오지 않는 프레임이라면 `num_persons=0`도 가능합니다. 이는 카메라 연결 실패나 산업안전 NG 확정이 아닙니다.

### 실습 28. 새 API를 직접 호출하고 DB 저장을 확인합니다

먼저 **PC 터미널**에서 최신 프레임 상태를 확인합니다. Base64로 인코딩된 이미지가 매우 길 수 있으므로 필요한 메타데이터만 출력합니다.

```bash
curl --fail 'http://127.0.0.1:8000/api/v1/jetson/stream/latest'   | python3 -c 'import json,sys; d=json.load(sys.stdin); print("ready", d["ready"], "fps", d["measured_fps"], "runtime", d["pose"]["runtime"] if d["pose"] else None)'
```

이제 브라우저에서 **최근 프레임 DB 저장**을 누르거나, 터미널에서 다음처럼 직접 호출합니다.

```bash
curl --fail -X POST 'http://127.0.0.1:8000/api/v1/jetson/stream/save'   | python3 -m json.tool
```

정상 응답은 다음 **필드**를 포함합니다. 이벤트 번호·프레임 순번은 실제 실행마다 달라집니다.

```json
{
  "saved": true,
  "event_id": "jetson:실제_생성된_UUID",
  "frame_seq": 123,
  "decision": "UNKNOWN",
  "runtime": "tensorrt"
}
```

여기서 `123`은 **필드 모양을 설명하는 예시**이지 실제 장비에 고정된 숫자가 아닙니다. 같은 프레임을 다시 저장하면 `saved: false`가 될 수 있으며, 새 프레임이면 새로운 이벤트 ID가 생성됩니다.

기존 8일차 조회 API에서 저장 결과를 확인합니다.

```bash
curl --fail 'http://127.0.0.1:8000/api/v1/jetson/events?limit=10'   | python3 -c 'import json,sys; a=json.load(sys.stdin); [print(x["event_id"],x["details"].get("runtime"),x["decision"]) for x in a]'
```

`runtime=tensorrt`, `decision=UNKNOWN`, `source_mode=live_inference`, `details.model_sha256` 등이 기록되었는지 확인합니다.

### 실습 29. PostgreSQL에 기존 기록과 새 기록이 함께 남아 있는지 확인합니다

**PC 터미널**에서 기존 DB 컨테이너를 통해 질의합니다. `.env`의 값을 Compose가 읽도록 실제 DB 사용자와 DB 이름을 `docker compose exec` 환경에서 사용합니다.

```bash
docker compose exec db sh -c   'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT event_id, task, source_mode, decision FROM inference_events WHERE event_id LIKE '''jetson:%''' ORDER BY id DESC LIMIT 10;"'
```

다른 날의 `recorded_result`, 6일차 Raspberry Pi, 8일차 GPU Pose 기록이 사라지지 않아야 합니다. 오늘 결과는 기존 테이블의 **새로운 `jetson:` 이벤트**이며, Pose만 수행했으므로 `decision`은 **`UNKNOWN`**입니다.

### 실습 30. 카메라·네트워크 장애를 실제로 확인합니다

PC 웹 화면을 계속 열어 둔 상태에서 아래 순서로 확인합니다.

1. 카메라가 정상 연결되어 `frame_seq`가 증가하는지 봅니다.
2. **허용된 시험 카메라에서만** IP Webcam 스트리밍을 잠시 중지하거나 USB 카메라를 안전하게 분리합니다.
3. 브라우저에 `reconnecting` 또는 오류 상태가 표시되는지 확인합니다.
4. 프레임이 더 이상 증가하지 않고 이전 값을 새로운 결과로 저장하지 않는지 확인합니다.
5. 카메라를 다시 연결하면 새 프레임이 나오고 `frame_seq`가 증가하는지 확인합니다.
6. 연결이 불안정하면 Jetson의 실제 서비스 로그를 확인합니다.

```bash
# Jetson SSH 터미널에서 실행합니다.
cd ~/ai_vision/subject13_ai_platform
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml logs --tail=100 pose
```

**GPU 모델 준비와 카메라 준비는 별개**입니다. `/health`가 `ready`인데 카메라가 `reconnecting`인 경우, TensorRT 엔진이 아니라 카메라 주소·장치 권한·네트워크가 원인일 수 있습니다. 장비가 불안정한 상황에서 사람의 안전을 판단하거나 경고 장치로 사용하지 않습니다.

---

## PART 7. 자동 테스트·문제 해결·일차별 작업 정리

### 실습 31. 기존 코드와 TensorRT 연결 API를 자동 테스트합니다

새로운 자동 테스트는 **Jetson GPU가 없는 PC에서도** JSON 계약, 중복 저장, GPU 응답 모드 구분, 카메라 준비 실패 시 API 동작 등을 확인합니다. 실제 Jetson TensorRT 추론 성공을 대신하는 테스트는 아닙니다.

**의사코드**

```text
8일차 pytorch_cuda 응답 유지 확인
                ↓
9일차 tensorrt 응답 수용 확인
                ↓
Keypoint·사람 수·JPEG 검증 오류 확인
                ↓
최신 프레임 조회 API 확인
                ↓
동일 event_id 저장 중복 방지 확인
                ↓
프레임 없음 → DB 저장 거절 확인
                ↓
네트워크 장애 → HTTP 503 확인
```

**파일: `tests/test_day09.py`**

```python
"""9일차: 모델 없는 PC에서 수행하는 API 규격·DB·장애 처리 단위 테스트."""
import base64
import io
import sys
import uuid
from pathlib import Path

import httpx
import pytest
from fastapi import FastAPI
from fastapi.testclient import TestClient
from PIL import Image
from sqlalchemy import create_engine
from sqlalchemy.orm import Session
from sqlalchemy.pool import StaticPool

from app.db import Base, get_db
from app.day09_api import LatestFrame, router
from app.jetson_api import PoseResponse, pose_to_event


def pose_json(persons=1, runtime="tensorrt"):
    people = [{"bbox_xyxy": [1, 2, 90, 92], "bbox_confidence": .92,
               "keypoints": [[float(i),float(i + 1),.9] for i in range(17)]} for _ in range(persons)]
    return {"event_id": str(uuid.uuid4()), "recorded_at": "2026-10-17T08:00:00+00:00",
            "input_ref": "day09_camera_frame.jpg", "width": 128, "height": 96,
            "num_persons": persons, "persons": people, "inference_ms": 32.1,
            "device_id": "jetson-orin-nano-super", "model_id": "yolo11n-pose",
            "model_sha256": "b" * 64, "runtime": runtime, "gpu_name": "Jetson GPU",
            "note": "Pose only"}


def jpeg_base64():
    b = io.BytesIO()
    Image.new("RGB", (128,96)).save(b, "JPEG")
    return base64.b64encode(b.getvalue()).decode("ascii")


def payload(ready=True, persons=1):
    return {"ready": ready, "camera_status": "running" if ready else "reconnecting",
            "measured_fps": 7.1 if ready else 0.0, "frames_total": 100,
            "last_error": None if ready else "camera disconnected",
            "pose": pose_json(persons) if ready else None,
            "jpeg_base64": jpeg_base64() if ready else None,
            "frame_seq": 100 if ready else None}


@pytest.fixture()
def client(monkeypatch):
    engine = create_engine("sqlite://", connect_args={"check_same_thread": False}, poolclass=StaticPool)
    Base.metadata.create_all(engine)
    app = FastAPI()
    app.include_router(router)
    def db_override():
        with Session(engine) as session:
            yield session
    app.dependency_overrides[get_db] = db_override
    return TestClient(app)


def fake_latest(monkeypatch, data):
    import app.day09_api as module
    monkeypatch.setattr(module, "fetch_latest", lambda: LatestFrame.model_validate(data))


def test_pose_adapter_tensorrt_is_unknown():
    event = pose_to_event(PoseResponse.model_validate(pose_json()))
    assert event.source_mode == "live_inference"
    assert event.decision == "UNKNOWN"
    assert event.reasons == ["POSE_ONLY_NO_ACTION_DECISION"]
    assert event.processor_version.startswith("day09-trt-")
    assert event.source_path.endswith(".engine")


def test_day08_cuda_still_supported():
    event = pose_to_event(PoseResponse.model_validate(pose_json(runtime="pytorch_cuda")))
    assert event.processor_version.startswith("day08-cuda-")
    assert event.source_path.endswith(".pt")


def test_keypoint_count_rejected():
    raw = payload()
    raw["pose"]["persons"][0]["keypoints"] = [[1, 2, .9]]
    with pytest.raises(ValueError):
        LatestFrame.model_validate(raw)


def test_inconsistent_people_rejected():
    raw = payload()
    raw["pose"]["num_persons"] = 0
    with pytest.raises(ValueError):
        LatestFrame.model_validate(raw)


def test_wrong_runtime_rejected():
    raw = payload()
    raw["pose"]["runtime"] = "pytorch_cuda"
    with pytest.raises(ValueError):
        LatestFrame.model_validate(raw)


def test_invalid_image_rejected():
    raw = payload()
    raw["jpeg_base64"] = "YWJjZA=="
    with pytest.raises(ValueError):
        LatestFrame.model_validate(raw)


def test_api_latest_and_page(client, monkeypatch):
    fake_latest(monkeypatch, payload())
    response = client.get("/api/v1/jetson/stream/latest")
    assert response.status_code == 200
    assert response.json()["ready"] is True
    assert client.get("/jetson-stream").status_code == 200


def test_save_latest_to_existing_table_and_repeat(client, monkeypatch):
    record = payload()
    fake_latest(monkeypatch, record)
    first = client.post("/api/v1/jetson/stream/save")
    second = client.post("/api/v1/jetson/stream/save")
    assert first.status_code == second.status_code == 200
    assert first.json()["saved"] is True
    assert second.json()["saved"] is False
    assert first.json()["event_id"] == second.json()["event_id"]


def test_no_frame_does_not_write_db(client, monkeypatch):
    fake_latest(monkeypatch, payload(ready=False))
    response = client.post("/api/v1/jetson/stream/save")
    assert response.status_code == 503


def test_no_detection_does_not_become_ng(client, monkeypatch):
    record = payload(persons=0)
    fake_latest(monkeypatch, record)
    res = client.post("/api/v1/jetson/stream/save")
    assert res.status_code == 200
    assert res.json()["decision"] == "UNKNOWN"


def test_remote_timeout_has_503(client, monkeypatch):
    import app.day09_api as module
    monkeypatch.setattr(module, "jetson_connection", lambda: ("http://jetson", "X"*32))
    class FakeClient:
        def __init__(self, **kwargs): pass
        def __enter__(self): return self
        def __exit__(self, *args): pass
        def get(self, *args, **kwargs):
            raise httpx.ConnectError("offline")
    monkeypatch.setattr(module.httpx, "Client", FakeClient)
    assert client.get("/api/v1/jetson/stream/latest").status_code == 503


def test_utils_camera_source_and_fps():
    import importlib.util
    path = Path(__file__).resolve().parents[1] / "deploy/day08_jetson/day09_utils.py"
    spec = importlib.util.spec_from_file_location("day09_utils_test", path)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    assert module.parse_camera_source("0") == 0
    assert module.parse_camera_source("http://192.168.0.20:8080/video") == "http://192.168.0.20:8080/video"
    with pytest.raises(ValueError):
        module.parse_camera_source("/etc/passwd")
    tracker = module.FrameRate()
    assert tracker.tick(now=10.0) == 0
    assert tracker.tick(now=10.2) == 5.0
```

실습 31의 테스트 파일을 저장했다면 **PC 프로젝트 최상위 폴더**에서 실행합니다. 가상환경 방식으로 검증한다면 1~6일차의 `.venv`를 활성화하고, 7일차 Docker 이미지 방식으로 검증한다면 API 컨테이너에서 실행합니다.

```bash
# PC에서 .venv 테스트
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m pytest -q tests/test_day08.py tests/test_day09.py

# 또는 PC Docker Compose 컨테이너에서 확인
# (새 테스트 파일이 이미 Docker 이미지에 포함되어 있어야 합니다.)
docker compose exec api python -m pytest -q tests/test_day08.py tests/test_day09.py
```

**시험 환경 검증:** 이 수업을 작성하면서 **8일차 기존 12개 + 9일차 신규 12개 = 총 24개 자동 테스트**가 통과했습니다. 다만 테스트는 실제 Jetson이 아닌 PC의 **HTTP·DB 테스트 대역**을 사용합니다. Jetson의 TensorRT 엔진 생성·실측 FPS·카메라 연결은 **실습 16~21과 27~30에서 실제 장치로 직접 확인**해야 합니다.

1~7일차의 테스트도 이후 함께 실행할 수 있습니다. 단, 해당 일차의 모델·데이터·서비스가 요구되는 통합 테스트는 준비물을 갖춘 상태에서 실행해야 합니다.

### 실습 31-1. 문제가 발생하면 기존 8일차 Pose 이미지로 복귀합니다

TensorRT 엔진 생성에 실패했거나 9일차 카메라 서비스가 시작하지 못했다면 **8일차에 정상 실행했던 `.pt` GPU 서비스로 되돌린 후** 오류를 해결할 수 있습니다. DB 초기화는 필요하지 않습니다.

**파일: `deploy/day08_jetson/compose.rollback.yaml`**

```yaml
services:
  pose:
    image: subject13-jetson-pose:day08
```

다음 명령은 **Jetson 터미널**에서만 실행합니다.

```bash
docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml stop pose

docker compose --env-file .env.jetson \
  -f deploy/day08_jetson/compose.jetson.yaml   -f deploy/day08_jetson/compose.rollback.yaml   up -d --no-build --no-deps pose
```

이때 **8일차 Docker 이미지 태그 `subject13-jetson-pose:day08`이 남아 있어야 합니다.** 이미지가 없다면 무작정 새 버전을 8일차 이미지로 가장하지 말고 기존 8일차 Dockerfile/이미지로 복구합니다. 복구 확인에는 `/health`의 `runtime=pytorch_cuda`를 사용합니다. 9일차로 복귀할 때는 복구용 override 파일을 빼고 기본 Compose로 `up -d --no-deps pose`를 실행합니다.

### 실습 32. 자주 발생하는 오류를 단계별로 확인합니다

| 오류·증상 | 먼저 확인할 곳 | 해결 방법 |
|---|---|---|
| `ModuleNotFoundError: tensorrt` | Jetson GPU Docker 이미지·JetPack 호환성 | JetPack과 TensorRT Python 바인딩이 포함된 GPU 이미지 사용 여부 확인 |
| TensorRT export 실패 / 메모리 부족 | 실제 Jetson 메모리·빌더 로그 | 다른 GPU 작업을 정리하고 엔진 빌드를 다시 수행; 임의 FP32 대체 금지 |
| `.engine` 파일 없음 | `artifacts/day09/jetson/` | 실습 16의 실제 엔진 생성부터 다시 확인 |
| `engine SHA256 ... 일치하지 않음` | `engine_manifest.json` | 올바른 장치에서 엔진·Manifest 다시 생성 |
| `CUDA False` | NVIDIA Runtime / JetPack | 8일차 GPU Docker 환경 확인 |
| 컨테이너 시작 직후 종료 | `docker compose logs pose` | 엔진 경로·Manifest·GPU Warm-up 확인 |
| 카메라 `reconnecting` | IP Webcam 주소·USB 장치 | Jetson에서 실제 URL 접근/`/dev/video*` 확인 |
| `ready false` | 카메라 연결·최신 프레임 시각 | 상태 확인 후 재연결을 기다림; 과거 프레임 재사용 금지 |
| FPS가 낮거나 불규칙함 | Jetson 추론 ms·GPU 동시 작업·카메라 해상도 | 동일 조건 재측정, 백그라운드 GPU 작업 정리, 입력 크기 확인 |
| PC API HTTP 503 | PC→Jetson LAN / API 키 / 카메라 프레임 | PC `.env` 및 Jetson `/health`, `/stream/latest` 확인 |
| PC API HTTP 502 | Jetson 응답 JSON 규격 | `runtime=tensorrt`, COCO17, 모델 SHA256 검증 |
| DB 저장 `saved:false` | 같은 `event_id`가 이미 저장됨 | 중복 저장 방지 정상 여부 확인 |
| 웹 화면이 열리지 않음 | PC 이미지 재빌드 여부 | `docker compose up -d --build api` 다시 실행 |

문제가 발생했다면 **엔진 → 이미지 한 장 → 카메라 최신 프레임 → PC API → 웹 화면 → DB 저장** 순서로 확인합니다. 어느 단계의 실패인지 확인한 뒤 해당 부분만 수정합니다.

### 실습 33. 9일차 운영 문서를 작성합니다

이 문서는 장치에서 사용한 실제 설정·성능 결과와 재시작 방법을 기록하는 데 사용합니다. **인증 키·비밀번호·개인 이미지는 문서에 넣지 않습니다.**

**파일: `docs/day09_tensorrt_runbook.md`**

````markdown
# 교과 13 · 9일차 TensorRT 연속 카메라 운영 기록

## 사용 환경
- Jetson 장치: Orin Nano Super
- JetPack / L4T: 실제 명령 출력 기록
- TensorRT 버전: 실제 명령 출력 기록
- 모델: yolo11n-pose.pt → TensorRT FP16 engine
- 엔진 SHA256 앞 12자리: 실제 값 기록
- 카메라 유형: IP Webcam 또는 USB
- 실제 비밀번호·카메라 인증 URL은 기록하지 않음

## 모델 생성과 성능
- 엔진 Manifest: artifacts/day09/jetson/engine_manifest.json
- 벤치마크: artifacts/day09/jetson/benchmarks/benchmark.csv
- PT CUDA p50/p95/FPS: 실제 측정값 기록
- TensorRT p50/p95/FPS: 실제 측정값 기록
- 사람 검출 결과가 달랐는지: 실제 관찰 결과 기록

## PC 플랫폼 확인
- 브라우저: http://127.0.0.1:8000/jetson-stream
- Jetson 최신 프레임 API: /api/v1/jetson/stream/latest
- PC 저장 API: POST /api/v1/jetson/stream/save
- PostgreSQL: 기존 inference_events 테이블
- Pose 결과 판정: UNKNOWN (Action/risk 미판정)

## 장애 복구
1. Jetson의 /health와 TensorRT 엔진 Manifest 확인
2. 카메라 status · 장치 주소·USB 연결 확인
3. Jetson pose 컨테이너 로그 확인
4. PC API·PostgreSQL 연결 확인
5. 재연결 후 새 frame_seq 증가·DB 저장 확인

## 다음 실습
10일차에는 추론 시간·FPS·API 실패·카메라 장애를 모니터링 지표로 수집합니다.
````

### 실습 34. 기존 README와 시스템 구조도를 업데이트합니다

**수정 파일: `README.md`**

1~8일차 설명은 그대로 두고 **마지막에** 다음 내용을 추가합니다.

````markdown
## 9일차 — Jetson TensorRT Pose 최적화와 연속 카메라

- 5일차 YOLO11n-Pose PT·ONNX 확인 후 Jetson에서 TensorRT FP16 엔진 생성
- 실제 이미지 한 장으로 PyTorch CUDA·TensorRT 지연시간/FPS 비교
- Jetson IP Webcam 또는 USB 카메라 연속 추론
- PC /jetson-stream: Skeleton·BBox·카메라 상태·FPS 표시
- 선택한 최신 Pose 결과: 기존 PostgreSQL inference_events 저장
- 실제 Action 분류 및 산업안전 위험 판정은 아직 수행하지 않음
- 실행 절차: docs/day09_tensorrt_runbook.md
````

**수정 파일: `docs/architecture.md`**

기존 1~8일차 흐름 아래에 다음을 추가합니다.

````markdown
## 9일차 — TensorRT·연속 카메라 확장

Jetson IP Webcam/USB → OpenCV → YOLO11n-Pose TensorRT FP16
                                      ↓
                          최근 이미지·Pose·FPS
                                      ↓
                        PC FastAPI /jetson-stream
                                      ↓
             사용자가 저장한 Pose InferenceEvent
                                      ↓
                         기존 PostgreSQL

- 모델 엔진: artifacts/day09/jetson/ (Jetson 장치별)
- Jetson 서비스: deploy/day08_jetson/app_day09.py
- PC 중계: app/day09_api.py
- 웹 화면: web/jetson_stream.html
- 모델 성능 비교: benchmark_pose.py
- 10일차: Prometheus·Grafana 상태·성능 모니터링으로 연결
````

### 실습 35. Git 제외 규칙을 확인합니다

`.gitignore`에 8일차 설정값을 보존한 상태로 다음 규칙이 있는지 확인하고, 없다면 추가합니다. 실제 `.engine`과 측정자료·사람 이미지는 공개 저장소에 올리지 않습니다.

```gitignore
# 9일차 장치별 TensorRT 엔진·실측 원본·카메라 설정
artifacts/day09/
*.engine
.env.jetson
deploy/day08_jetson/samples/
```

특히 PC의 `.env`, Jetson의 `.env.jetson`에 있는 API 키를 Git으로 전송하지 않습니다. 이미 Git이 추적하는 파일은 `.gitignore` 추가만으로 자동 제외되지 않으므로 **`git status`와 `git ls-files`**로 확인합니다.

---

# 6. 오늘 작성한 내용의 최종 정리 및 GitHub 업로드

## 6.1. 오늘 완성한 것을 정리합니다

8일차에는 **Jetson에서 실제 YOLO11n-Pose 정지 이미지 추론**을 실행했습니다. 오늘은 같은 가중치를 **TensorRT FP16 엔진으로 최적화**하고, 실제 카메라의 연속 프레임을 처리하도록 확장했습니다. 기존 7일차 Docker Compose와 2일차 PostgreSQL은 계속 사용했습니다.

```text
[8일차]   JPG/PNG → Jetson .pt CUDA → 사람 Pose → PC DB
                                   ↓
[9일차]   같은 PT → Jetson TensorRT FP16 엔진
                                   ↓
              실제 이미지 성능 비교 (p50/p95/FPS)
                                   ↓
             IP Webcam/USB 연속 프레임
                                   ↓
                   Jetson TensorRT Pose
                                   ↓
              최신 JPEG + COCO17 + FPS
                                   ↓
                PC FastAPI + 웹 Skeleton
                                   ↓
             선택한 결과만 PostgreSQL 저장
                                   ↓
              장애 재연결 확인 + Git 기록
```

오늘의 `UNKNOWN`은 **Pose만 검출했고 동작의 정상·위험 여부를 평가하지 않았다는 뜻**입니다. 사람이 검출되지 않거나 카메라가 끊겼다고 임의로 NG를 작성하지 않습니다. 실제 모델 성능이 좋아졌다는 결론도 **Jetson 측정값을 확인한 뒤** 내립니다.

## 6.2. 최종 자가 체크리스트

- [ ] 8일차 PC `api/db/redis/worker` 컨테이너와 Jetson GPU 환경을 확인했습니다.
- [ ] 5일차 `.pt`, `.onnx`, Manifest, 8일차 실제 사람 이미지를 확인했습니다.
- [ ] Jetson에서 TensorRT 엔진 파일과 `engine_manifest.json`을 생성하고 SHA256을 확인했습니다.
- [ ] 같은 이미지로 CUDA·TensorRT p50·p95·순차 FPS 측정 결과를 저장했습니다.
- [ ] 카메라 입력을 실제 Jetson에서 열고 연속 추론 결과를 확인했습니다.
- [ ] Jetson `/health`가 `runtime=tensorrt`와 GPU 준비 상태를 보여줍니다.
- [ ] PC `/jetson-stream`에 최신 이미지·BBox·Skeleton·FPS가 표시됩니다.
- [ ] 최근 프레임을 기존 PostgreSQL에 저장하고 `runtime=tensorrt`를 확인했습니다.
- [ ] 카메라 연결을 끊고 다시 연결하여 상태와 최신 프레임 복구를 확인했습니다.
- [ ] 8일차·9일차 자동 테스트가 통과했습니다.
- [ ] README·구조도·운영 문서를 업데이트했습니다.
- [ ] 인증 키·원본 사람 이미지·TensorRT 엔진이 Git에 올라가지 않는지 확인했습니다.

## 6.3. 오늘 작성한 소스코드를 Git에 기록합니다

**PC의 기존 교과 13 프로젝트**에서 실행합니다. 새 저장소를 만들거나 지난 1~8일차의 Git 기록을 초기화하지 않습니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git status

git add app/day09_api.py app/jetson_api.py app/main.py \
  deploy/day08_jetson/Dockerfile.jetson \
  deploy/day08_jetson/compose.jetson.yaml \
  deploy/day08_jetson/compose.rollback.yaml \
  deploy/day08_jetson/.env.jetson.example \
  deploy/day08_jetson/day09_utils.py \
  deploy/day08_jetson/export_tensorrt.py \
  deploy/day08_jetson/benchmark_pose.py \
  deploy/day08_jetson/app_day09.py \
  web/jetson_stream.html tests/test_day09.py \
  docs/day09_tensorrt_runbook.md docs/architecture.md README.md .gitignore

git status
```

`artifacts/day09/`, 원본 이미지, `.env`, `.env.jetson`이 **Staged 목록에 없는지** 확인합니다. USB 카메라 실습으로 `deploy/day08_jetson/compose.usb.yaml`을 작성했다면 검토 후 함께 추가할 수 있습니다. `compose.rollback.yaml`도 실제로 작성해 Git에 포함합니다.

```bash
git commit -m "교과13 9일차: Jetson TensorRT 최적화 및 연속 Pose 서비스 연결"
git log -1 --oneline
git status
```

### 외부 GitHub 업로드가 허용된 경우에만 진행합니다

```bash
git remote -v
git push origin main
```

기존 Git의 원격 저장소·브랜치를 그대로 사용합니다. 실제 기본 브랜치가 `main`이 아니라면 현재 브랜치를 `git branch --show-current`로 확인해 맞게 Push합니다. 폐쇄망에서는 외부 GitHub 대신 승인된 내부 Git 서버 또는 로컬 Git만 사용합니다.

## 6.4. 10일차와 연결합니다

10일차에는 오늘부터 실제로 측정한 **TensorRT 추론 시간, 연속 처리 FPS, 카메라 연결 상태, PC FastAPI 요청·실패 횟수**를 Prometheus 지표로 수집하고 Grafana에서 시간에 따른 변화를 확인합니다. **9일차의 추론·DB·웹 코드를 새로 만드는 것이 아니라** 같은 서비스에 관측 기능을 추가합니다.

---

## 기술 문서 참고

- Ultralytics YOLO 내보내기: https://docs.ultralytics.com/modes/export/
- Ultralytics TensorRT 통합: https://docs.ultralytics.com/integrations/tensorrt/
- Ultralytics NVIDIA Jetson 가이드: https://docs.ultralytics.com/guides/nvidia-jetson/

이 링크는 **TensorRT 옵션·Jetson 호환성에 대한 참고 문서**입니다. 실제 사용할 엔진 파일·CUDA 사용 상태·FPS 결과는 반드시 본인의 Jetson 장치에서 검증합니다.

