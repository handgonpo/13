## 1. 오늘의 목표

9일차에는 Jetson 카메라에서 영상을 받아 YOLO11n-Pose TensorRT 엔진으로 추론하고, PC의 FastAPI 웹 화면에서 실측 FPS와 Skeleton을 확인했습니다. 그러나 화면이 정상으로 보이더라도 API가 느려지거나, Jetson 연결이 중단되거나, 최신 카메라 프레임이 더 이상 생성되지 않는 문제는 따로 감시해야 합니다.

오늘은 FastAPI와 Jetson에서 실제 측정한 지표를 **Prometheus로 수집**하고 **Grafana에 그래프로 표시**합니다. 수업 종료 시 다음을 직접 확인할 수 있어야 합니다.

- 7일차 Docker Compose의 FastAPI·PostgreSQL·Redis·Celery를 유지하면서 Prometheus·Grafana를 추가합니다.
- FastAPI 요청 수, 응답시간과 상태 코드를 `/metrics`에서 확인합니다.
- Jetson의 카메라 연결 상태, 실측 FPS, 추론시간, 누적 오류 수를 안전한 인증 API에서 읽습니다.
- Prometheus Targets와 Grafana 대시보드를 확인하고 실제 장애 상황에서 지표가 어떻게 바뀌는지 관찰합니다.
- 기존 9일차 추론 화면 및 PostgreSQL 저장 기능이 변함없이 작동하는지 재확인합니다.
- 자동 테스트, 운영 기록, Git 커밋으로 10일차를 마무리합니다.

**오늘의 완성 결과:** `http://127.0.0.1:8000/monitoring`에서 모니터링 안내 화면을 열고, Prometheus `:9090` 및 Grafana `:3000`에서 **실제 측정한 지표**를 확인합니다. 값은 장치 상태에 따라 다르므로 임의의 FPS·추론시간을 예상 정답으로 사용하지 않습니다.

## 2. 오늘 수업의 전체 흐름

![[Pasted image 20261018131910.png]]

```text
[7일차 PC Docker]                    [9일차 Jetson GPU Docker]
FastAPI + PostgreSQL                 YOLO11n-Pose TensorRT
Redis + Celery                       IP Webcam / USB 카메라
      │                                        │
      ├─ API 요청 수·응답시간                   └─ FPS·추론시간·프레임·오류
      │                                        │
      │                               인증된 Jetson Telemetry API
      │                                        │
      └──────── PC FastAPI /metrics ────────────┘
                         ↓
              Prometheus 주기적 수집
                         ↓
             Grafana Dashboard · Alert
                         ↓
       서비스 중단·카메라 중단·복구 확인
                         ↓
         기존 추론·DB 기록 회귀 테스트
                         ↓
                README · Git 정리
                         ↓
       [11일차] Cloud 배포 시 동일 지표 확장
```

## 3. 실습 전에 알아야 할 핵심

### 3.1. 1~9일차에서 오늘 이어받을 것

| 기존 일차 | 그대로 사용할 기능 | 확인할 파일·주소 |
|---|---|---|
| 2일차 | PostgreSQL 이벤트 이력 | `app/db.py`, `app/models.py`, `app/repository.py` |
| 3일차 | Redis·Celery 백그라운드 작업 | `app/celery_app.py`, `compose.yaml` |
| 7일차 | PC FastAPI·DB·Redis·Worker Compose | `Dockerfile`, `requirements.docker.txt`, `compose.yaml` |
| 8일차 | Jetson 인증·Pose 공통 이벤트 변환 | `app/jetson_api.py`, `app/jetson_repo.py` |
| 9일차 | TensorRT 연속 카메라와 PC 최신 화면 | `deploy/day08_jetson/app_day09.py`, `web/jetson_stream.html`, `app/day09_api.py` |

**이전 결과물이 없다면:** 실습에 앞서 해당 일차의 완료된 소스 파일과 Jetson 모델/엔진·Manifest, 환경설정, 기존 DB를 확보합니다. 교과 9·10 원본 모델을 오늘 새로 학습할 필요는 없습니다. **실제 Jetson TensorRT 엔진은 해당 Jetson 장치에서 9일차대로 만들어져 있어야** 실측 FPS가 나타납니다. Jetson이 아직 연결되지 않아도 PC FastAPI 모니터링은 진행할 수 있지만, Jetson 지표는 `up=0` 또는 미측정으로 표시합니다.

### 3.2. Prometheus와 Grafana의 역할

**Prometheus**는 HTTP `/metrics`에서 수치를 읽어 시간순으로 보관합니다. 이를 *Scrape*라고 합니다. **Grafana**는 Prometheus의 기록을 조회하여 그래프로 표시합니다. Grafana가 카메라나 AI 모델을 직접 실행하는 것은 아닙니다.

```text
FastAPI /metrics → Prometheus 수집·시계열 저장 → Grafana 시각화
```

### 3.3. 오늘 사용할 지표를 구분합니다

| 지표 | 의미 | 어떤 값을 조심해서 해석하나요? |
|---|---|---|
| `platform_http_requests_total` | FastAPI 누적 요청 수 | 프로세스 재시작 시 다시 시작하므로 `rate()` 사용 |
| `platform_http_request_duration_seconds` | API 요청 소요시간 히스토그램 | 모델 추론시간과 다름 |
| `platform_jetson_up` | PC가 Jetson 인증 API에 접근한 성공 여부 | `0`은 네트워크·인증·Jetson 중단 등 여러 원인 가능 |
| `platform_jetson_camera_ready` | 최근 3초 이내 프레임과 정상 카메라 상태 | `0`은 최신 프레임 없음 |
| `platform_jetson_processing_fps` | 9일차 Jetson 처리 루프의 실측 FPS | 카메라 원본 FPS나 웹 화면 FPS와 다름 |
| `platform_jetson_inference_last_ms` | 최근 Pose 모델 호출시간(ms) | 전체 처리 지연시간과 다름 |
| `platform_jetson_inference_recent_p95_ms` | 최근 최대 200회 추론시간의 p95 | 무한 기간의 p95가 아님 |
| `platform_jetson_errors_seen` | Jetson 프로세스 내 누적 오류 수 | 재시작하면 초기화 |

오늘은 **GPU 사용률(%)을 직접 수집하지 않습니다.** `platform_jetson_gpu_available`은 CUDA 사용 가능 여부(0/1)일 뿐 GPU 사용률이 아닙니다. `0ms`도 실제 추론이 0ms라는 보장은 없으며 **아직 측정값이 없는 초기 상태**일 수 있습니다.

### 3.4. 운영에서 조심할 것

- `/metrics`에 영상, 원본 이미지, Keypoint 배열, 비밀번호, 인증키를 넣지 않습니다.
- URL의 개별 `event_id`, 사용자 파일명 등을 Prometheus 라벨로 넣으면 지표 수가 폭증하므로 공통 API 경로만 기록합니다.
- Jetson의 `/api/v1/jetson/telemetry`는 기존 `X-Jetson-Key` 인증을 유지합니다.
- 오늘도 **OK/NG/UNKNOWN은 검사·동작 비교 결과**이고, 모니터링 경고는 서비스 운영 상태에 관한 것입니다.
- 수업 PC 외부로 Prometheus·Grafana 포트를 공개하지 않습니다. 원격 접속은 보안 정책에 맞게 구성합니다.

## 4. 오늘 사용할 기술 스택

| 도구 | 오늘의 역할 |
|---|---|
| Python · `prometheus-client` | FastAPI Counter·Histogram·Gauge 생성 |
| FastAPI · HTTPX | `/metrics` 및 Jetson 인증 Telemetry 조회 |
| Jetson TensorRT · OpenCV | 9일차 실제 추론과 카메라 측정값 제공 |
| Docker Compose | 기존 PC 서비스에 Prometheus·Grafana 추가 |
| Prometheus · PromQL | 지표 수집·질의·간단한 경보 조건 |
| Grafana | 데이터소스·대시보드 자동 구성 및 시각화 |
| Pytest | 계측·인증·API·Compose 구성 회귀 검증 |
| Git | 코드·설정·운영 문서 기록 |

# 5. 처음부터 끝까지 따라 하는 상세 실습

## PART 1. 9일차 프로젝트와 실행환경을 확인합니다

### 실습 1. 기존 교과 13 프로젝트를 다시 엽니다

**이번 실습은 무엇을 하나요?** 기존 플랫폼과 Jetson 코드를 그대로 사용하기 위해 현재 위치와 전날 실행 결과를 확인합니다. 새 프로젝트를 생성하지 않습니다.

**PC WSL 터미널:**

```bash
cd ~/ai_vision/subject13_ai_platform
pwd
ls compose.yaml Dockerfile requirements.docker.txt
ls app/main.py app/day09_api.py deploy/day08_jetson/app_day09.py
code .
```

정상이라면 `pwd` 끝에 `subject13_ai_platform`이 표시됩니다. VS Code와 터미널의 작업 경로가 동일한지 확인합니다.

### 실습 2. 오늘 사용할 결과물이 모두 있는지 자동으로 확인합니다

지난 수업 파일이 하나라도 빠지면 Docker 빌드 전에 먼저 복구하는 편이 안전합니다. 다음 파일을 작성합니다.

**파일: `scripts/20_check_day10_inputs.py`**

```python
"""10일차 시작 전 PC의 7·9일차 주요 결과물을 확인합니다."""
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]
NEEDED = [
    'compose.yaml', 'Dockerfile', 'requirements.docker.txt',
    'app/main.py', 'app/day09_api.py', 'app/jetson_api.py',
    'deploy/day08_jetson/compose.jetson.yaml',
    'deploy/day08_jetson/app_day09.py',
    'web/jetson_stream.html',
]
missing = [p for p in NEEDED if not (ROOT / p).is_file()]
for item in NEEDED:
    print(('없음' if item in missing else '확인'), item)
if missing:
    raise SystemExit(f'{len(missing)}개 파일이 없습니다. 이전 일차 산출물을 확인하세요.')
print('PC의 10일차 준비 파일 확인 완료')
```

이 스크립트는 **파일 존재 여부만 확인**합니다. 실제 GPU나 API 작동을 대신 검증하지 않습니다.

```bash
python scripts/20_check_day10_inputs.py
```

모든 파일에 `확인`이 출력되고 마지막에 `PC의 10일차 준비 파일 확인 완료`가 나오면 다음으로 진행합니다. 파일이 없으면 해당 일차 완료본에서 복구한 후 다시 실행합니다.

### 실습 3. PC와 Jetson의 현재 서비스 상태를 확인합니다

**PC 터미널:**

```bash
docker compose ps
curl -s http://127.0.0.1:8000/health | python3 -m json.tool
curl -s http://127.0.0.1:8000/api/v1/summary | python3 -m json.tool
```

**Jetson 터미널(SSH 접속 후):**

```bash
cd ~/subject13_ai_platform/deploy/day08_jetson
# 실제 장치의 교과 13 프로젝트 경로가 다르면 위 cd 경로만 수정합니다.
docker compose -f compose.jetson.yaml ps
curl -s http://127.0.0.1:8018/health | python3 -m json.tool
```

> 위 `8018` 포트는 전일 Jetson Compose의 실제 포트 매핑을 확인하여 일치할 때 사용합니다. 다른 포트라면 **전날 설정값으로 수정**합니다. 해당 장치가 아직 없다면 PC 실습만 먼저 진행하고 장치 관련 실습을 나중에 검증합니다.

## PART 2. FastAPI에 요청 수·응답시간을 기록합니다

### 실습 4. Prometheus 패키지를 추가합니다

**이번 코드는 무엇을 하나요?** 실행 중인 FastAPI가 지표를 기록하고 `/metrics`에서 노출할 수 있도록 라이브러리를 설치합니다.

기존 `requirements.docker.txt` 마지막에 다음 **한 줄을 추가**합니다. 기존 항목은 삭제하지 않습니다.

```text
prometheus-client>=0.20,<1
```

1~6일차 PC `.venv`에서도 테스트할 경우:

```bash
source .venv/bin/activate
python -m pip install 'prometheus-client>=0.20,<1'
```

**의사코드**

```text
HTTP 요청 시작
   ↓
요청 시작 시간 기록
   ↓
기존 FastAPI 처리 함수 실행
   ↓
응답 상태코드·소요시간 기록
   ↓
Prometheus Counter·Histogram에 누적
   ↓
GET /metrics 요청 시 텍스트 형식으로 반환
```

### 실습 5. FastAPI 계측 모듈을 작성합니다

파일을 새로 생성합니다. 메트릭 이름은 앞으로의 수업에서도 유지합니다.

**파일: `app/day10_metrics.py`**

```python
"""10일차: FastAPI + 인증된 Jetson 텔레메트리를 Prometheus 형식으로 변환."""
import os
from time import perf_counter

import httpx
from prometheus_client import Counter, Gauge, Histogram
from fastapi import FastAPI

HTTP_REQUESTS = Counter(
    "platform_http_requests_total", "PC FastAPI HTTP requests", ("method", "route", "status")
)
HTTP_DURATION = Histogram(
    "platform_http_request_duration_seconds", "PC FastAPI request duration",
    ("method", "route"), buckets=(.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10)
)
JETSON_UP = Gauge("platform_jetson_up", "Jetson telemetry connection is available")
JETSON_CAMERA_READY = Gauge("platform_jetson_camera_ready", "Recent Jetson camera frame available")
JETSON_GPU_AVAILABLE = Gauge("platform_jetson_gpu_available", "CUDA available on Jetson")
JETSON_FPS = Gauge("platform_jetson_processing_fps", "Jetson reported processed frames per second")
JETSON_INFERENCE_LAST_MS = Gauge("platform_jetson_inference_last_ms", "Last Jetson TensorRT Pose inference duration, ms")
JETSON_INFERENCE_P95_MS = Gauge("platform_jetson_inference_recent_p95_ms", "Recent <=200 Pose inference samples p95, ms")
JETSON_ERROR_COUNT = Gauge("platform_jetson_errors_seen", "Jetson errors since process startup; resets on restart")
JETSON_FRAMES_COUNT = Gauge("platform_jetson_frames_seen", "Jetson camera frames since process startup")


def install_metrics(app: FastAPI) -> None:
    """애플리케이션에 한 번만 설치합니다. 메트릭 스크랩 자체는 제외합니다."""
    @app.middleware("http")
    async def observe_request(request, call_next):
        if request.url.path == "/metrics":
            return await call_next(request)
        started = perf_counter()
        status = 500
        try:
            response = await call_next(request)
            status = response.status_code
            return response
        finally:
            duration = perf_counter() - started
            route = getattr(request.scope.get("route"), "path", "unmatched")
            # 실제 URL의 event_id나 사용자 입력을 라벨로 넣지 않습니다.
            HTTP_REQUESTS.labels(request.method, route, str(status)).inc()
            HTTP_DURATION.labels(request.method, route).observe(duration)


def request_jetson_telemetry(url: str, key: str) -> dict:
    with httpx.Client(timeout=2.0, trust_env=False) as client:
        response = client.get(url + "/api/v1/jetson/telemetry", headers={"X-Jetson-Key": key})
        response.raise_for_status()
        return response.json()


def poll_jetson_telemetry() -> bool:
    """Prometheus의 10초 간격 scrape 시 PC에서 Jetson의 인증 API를 짧게 조회합니다."""
    url = os.getenv("DAY08_JETSON_URL", "").rstrip("/")
    key = os.getenv("DAY08_JETSON_API_KEY", "")
    if not url.startswith("http://") or len(key) < 24:
        JETSON_UP.set(0)
        JETSON_CAMERA_READY.set(0)
        JETSON_FPS.set(0)
        return False
    try:
        item = request_jetson_telemetry(url, key)
        # 예상한 JSON 형식이 아니면 연결 실패로 간주합니다.
        ready = item["camera_ready"]
        gpu = item["gpu_available"]
        fps = float(item["measured_fps"])
        frames = int(item["frames_total"])
        errors = int(item["error_count"])
        if not isinstance(ready, bool) or not isinstance(gpu, bool):
            raise ValueError("잘못된 준비 상태")
        if not 0 <= fps <= 200 or frames < 0 or errors < 0:
            raise ValueError("실측 수치 범위가 잘못되었습니다.")
        last = item.get("inference_last_ms")
        p95 = item.get("inference_p95_ms")
        for sample in (last, p95):
            if sample is not None and not 0 <= float(sample) <= 120000:
                raise ValueError("잘못된 추론시간")
        JETSON_GPU_AVAILABLE.set(int(gpu))
        JETSON_CAMERA_READY.set(int(ready))
        JETSON_FPS.set(fps if ready else 0)
        JETSON_FRAMES_COUNT.set(frames)
        JETSON_ERROR_COUNT.set(errors)
        # 아직 추론이 없으면 0으로 표시하지만 미측정임을 화면에서 따로 설명합니다.
        JETSON_INFERENCE_LAST_MS.set(float(last) if last is not None else 0)
        JETSON_INFERENCE_P95_MS.set(float(p95) if p95 is not None else 0)
        JETSON_UP.set(1)
        return True
    except (httpx.HTTPError, ValueError, KeyError, TypeError, OverflowError):
        JETSON_UP.set(0)
        JETSON_CAMERA_READY.set(0)
        JETSON_FPS.set(0)
        JETSON_GPU_AVAILABLE.set(0)
        JETSON_INFERENCE_LAST_MS.set(0)
        JETSON_INFERENCE_P95_MS.set(0)
        return False
```

**코드 이해하기**

`Counter`는 요청 발생 횟수를 누적하고, `Histogram`은 요청시간의 분포를 구간별로 저장합니다. `Gauge`는 Jetson 연결 성공 여부·FPS처럼 매번 달라지는 현재 상태를 표시합니다.

`install_metrics()`는 모든 HTTP 요청을 측정하되 Prometheus 자신이 호출하는 `/metrics`는 제외합니다. `request.scope['route']`에서 얻는 **라우트 템플릿**을 사용하므로 작업별 `event_id`를 지표 라벨로 만들지 않습니다.

`poll_jetson_telemetry()`는 Jetson에서 받은 값이 정상 범위인지 확인한 후 Gauge를 갱신합니다. Jetson이 꺼져 있거나 인증이 실패하면 `platform_jetson_up=0`으로 기록합니다. **연결 실패를 임의의 정상 값으로 바꾸지 않습니다.**

### 실습 6. `/metrics`와 모니터링 안내 페이지를 추가합니다

파일을 새로 만듭니다.

**파일: `app/day10_api.py`**

```python
"""10일차: Prometheus scrape와 모니터링 안내 페이지."""
from pathlib import Path

from fastapi import APIRouter
from fastapi.responses import FileResponse, Response
from prometheus_client import CONTENT_TYPE_LATEST, generate_latest

from app.day10_metrics import poll_jetson_telemetry

router = APIRouter(tags=["day10-monitoring"])
ROOT = Path(__file__).resolve().parents[1]


@router.get("/metrics", include_in_schema=False)
def metrics():
    # 실패해도 /metrics는 200으로 제공하며 platform_jetson_up=0을 내보냅니다.
    poll_jetson_telemetry()
    return Response(content=generate_latest(), media_type=CONTENT_TYPE_LATEST)


@router.get("/monitoring", include_in_schema=False)
def monitoring_home():
    return FileResponse(ROOT / "web" / "monitoring.html")
```

**코드 이해하기:** `/metrics`가 호출될 때 PC는 Jetson의 작은 JSON Telemetry API를 짧게 조회한 다음 Prometheus 텍스트를 반환합니다. 카메라 이미지 자체를 이 API로 전달하지 않습니다.

### 실습 7. 기존 `app/main.py`에 10일차 연결 코드를 추가합니다

1~9일차의 기존 라우터를 **삭제하지 않습니다.** `app/main.py`에 다음 import가 없다면 추가합니다.

```python
from app.day10_api import router as day10_router
from app.day10_metrics import install_metrics
```

기존에 `app = FastAPI(...)`를 생성하는 코드를 찾아 **그 아래**에 다음 두 줄을 추가합니다.

```python
install_metrics(app)
app.include_router(day10_router)
```

이미 `app.include_router(day09_router)` 등이 있다면 그대로 두고 새 라우터만 추가합니다. `install_metrics(app)`를 여러 번 실행하지 않도록 **한 번만 작성**합니다.

**확인:** PC `.venv`에서 실행 가능한 경우 다음을 실행합니다.

```bash
python -m compileall -q app/day10_metrics.py app/day10_api.py app/main.py
```

## PART 3. Jetson에서 실제 FPS·추론시간·오류를 읽습니다

### 실습 8. Jetson 텔레메트리 기록 클래스를 추가합니다

이 파일은 **Jetson에 배포하는 코드**입니다. 기존 9일차 `app_day09.py` 옆에 추가합니다.

**의사코드**

```text
Jetson TensorRT 모델 호출
      ↓
추론 소요시간(ms) 추가
      ↓
최근 최대 200개 추론시간 보관
      ↓
카메라 또는 추론 오류 발생 시 오류 횟수 증가
      ↓
요청 시 마지막 시간·p95·누적 오류를 작은 JSON으로 제공
```

**파일: `deploy/day08_jetson/day10_telemetry.py`**

```python
"""10일차: Jetson의 실측 추론시간·오류 누적값을 안전하게 보관합니다.

GPU 사용률을 추정하지 않습니다. 카메라/FPS는 9일차 STATE에서 읽습니다.
"""
from collections import deque
from threading import Lock


class PoseTelemetry:
    def __init__(self):
        self._lock = Lock()
        self._latencies = deque(maxlen=200)
        self._inference_count = 0
        self._error_count = 0

    def record_inference(self, milliseconds: float) -> None:
        if milliseconds < 0:
            raise ValueError("추론 시간은 음수가 될 수 없습니다.")
        with self._lock:
            self._inference_count += 1
            self._latencies.append(float(milliseconds))

    def record_error(self) -> None:
        with self._lock:
            self._error_count += 1

    def snapshot(self) -> dict:
        with self._lock:
            ordered = sorted(self._latencies)
            # 관찰값의 최근 최대 200개에 대한 nearest-rank p95입니다.
            p95 = ordered[max(0, (95 * len(ordered) + 99) // 100 - 1)] if ordered else None
            return {
                "inference_count": self._inference_count,
                "inference_last_ms": self._latencies[-1] if self._latencies else None,
                "inference_p95_ms": p95,
                "error_count": self._error_count,
            }
```

`deque(maxlen=200)`은 오래된 값을 계속 메모리에 쌓지 않고 최근 200개만 보관합니다. 계산한 p95는 **최근 최대 200개 표본**의 95백분위이며 운영환경 전체의 누적 p95가 아닙니다.

### 실습 9. 기존 TensorRT 추론 파일에 계측을 연결합니다

**파일 수정: `deploy/day08_jetson/app_day09.py`**

9일차 코드를 새로 쓰는 것이 아니라 아래 위치만 찾아 추가합니다.

**① import 부분에 추가**

```python
from day10_telemetry import PoseTelemetry

POSE_TELEMETRY = PoseTelemetry()
```

**② 기존 `infer_frame()` 안에서 `elapsed_ms`를 계산한 바로 다음 줄에 추가**

```python
POSE_TELEMETRY.record_inference(elapsed_ms)
```

**③ 기존 `camera_loop()`의 카메라 열기 실패·프레임 읽기 실패·추론 예외 분기에서** 각 오류 발생 시 다음 코드를 한 번씩 실행합니다. 기존 재연결 처리는 유지합니다.

```python
POSE_TELEMETRY.record_error()
```

동일한 오류를 한 번 처리하면서 여러 차례 호출해 중복 집계하지 않도록 주의합니다. `record_error()`는 정상 프레임마다 실행하는 코드가 아닙니다.

### 실습 10. 인증된 Jetson 상태 조회 API를 추가합니다

같은 `app_day09.py`의 다른 `@app.get(...)` 아래에 다음 코드를 추가합니다. 이 API는 **FPS·카메라 상태·추론시간**만 반환하며, 영상 JPEG나 Keypoint 배열을 반환하지 않습니다.

```python
@app.get("/api/v1/jetson/telemetry")
def telemetry(x_jetson_key: str | None = Header(default=None, alias="X-Jetson-Key")):
    check_key(x_jetson_key)
    with STATE_LOCK:
        status = STATE["camera_status"]
        fps = float(STATE["measured_fps"])
        count = int(STATE["frames_total"])
        latest_item = STATE["latest"]
        is_fresh = (latest_item is not None and
                    monotonic() - latest_item["created_monotonic"] <= 3.0)
    return {
        "camera_status": status,
        "camera_ready": bool(status == "running" and is_fresh),
        "measured_fps": fps if is_fresh else 0.0,
        "frames_total": count,
        "gpu_available": bool(torch.cuda.is_available()),
        **POSE_TELEMETRY.snapshot(),
    }
```

**코드 이해하기:** `STATE`는 9일차의 실제 카메라 상태를 그대로 읽습니다. `is_fresh`가 거짓이면 오래된 이미지를 최신 프레임으로 보지 않습니다. 인증키를 모르거나 잘못 입력하면 HTTP 401을 반환합니다.

### 실습 11. Jetson Docker에 새 파일을 포함합니다

**파일 수정: `deploy/day08_jetson/Dockerfile.jetson`**

기존 `COPY day09_utils.py ...` 등의 행을 확인하고, 같은 작업 폴더에 새 파일이 들어가도록 아래와 같은 `COPY` 구문을 **추가**합니다. 9일차 Dockerfile의 `WORKDIR /jetson_app`에 맞춰 사용합니다.

```dockerfile
COPY day10_telemetry.py /jetson_app/day10_telemetry.py
```

> Jetson 기존 Dockerfile의 `WORKDIR`와 `COPY app_day09.py ...` 목적지가 `/jetson_app`이 아닌 경우, **기존 `app_day09.py`를 놓는 폴더와 동일한 위치**로 지정해야 합니다. Python이 `from day10_telemetry import PoseTelemetry`를 읽을 수 있어야 합니다.

다음 단계에서 이미지를 다시 빌드하면 새 코드가 컨테이너에 포함됩니다.

## PART 4. PC에 Prometheus·Grafana를 추가합니다

### 실습 12. 모니터링 폴더를 생성합니다

**PC 프로젝트 최상위**에서 진행합니다.

```bash
mkdir -p monitoring/grafana/provisioning/datasources
mkdir -p monitoring/grafana/provisioning/dashboards
mkdir -p monitoring/grafana/dashboards
```

이 폴더는 7일차 DB 데이터나 9일차 TensorRT 엔진 폴더를 대체하지 않습니다.

### 실습 13. Prometheus 수집 설정을 만듭니다

**의사코드**

```text
10초마다 PC FastAPI /metrics 읽기
         ↓
PC의 API 지표와 Jetson에서 조회한 지표 기록
         ↓
규칙 파일의 서비스 연결·카메라 상태 조건 평가
         ↓
시계열 데이터로 저장
```

**파일: `monitoring/prometheus.yml`**

```yaml
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
        labels:
          system: 'subject13_pc'

  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
```

`api:8000`은 7일차 Compose 네트워크에 속한 **서비스 이름**입니다. Prometheus 컨테이너 안에서 `127.0.0.1:8000`으로 바꾸면 PC API가 아닌 자기 컨테이너를 가리키므로 잘못된 설정입니다.

### 실습 14. 경고 규칙을 작성합니다

오늘은 Alertmanager까지 구축하지 않습니다. Prometheus의 **경고 조건이 충족되는지** 확인합니다. 알림 메시지를 실제 휴대전화나 메일로 보내는 기능은 아직 없습니다.

**파일: `monitoring/alert_rules.yml`**

```yaml
groups:
  - name: subject13_practice
    rules:
      - alert: PlatformApiUnavailable
        expr: up{job="platform_api"} == 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "실습용 FastAPI 메트릭 수집 실패"
      - alert: JetsonTelemetryUnavailable
        expr: platform_jetson_up{job="platform_api"} == 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Jetson 텔레메트리 접근 실패 또는 인증/설정 누락"
      - alert: JetsonCameraUnavailable
        expr: platform_jetson_up{job="platform_api"} == 1 and platform_jetson_camera_ready{job="platform_api"} == 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Jetson 연결은 되지만 최신 카메라 프레임이 없음"
```

`for: 1m`은 조건이 한 순간 발생했다고 즉시 경고로 확정하는 것이 아니라, **1분간 유지되어야 firing 상태가 되는 것**을 뜻합니다. Jetson 전원을 잠깐 다시 켜는 경우 등을 구분하는 데 도움이 됩니다.

### 실습 15. Grafana Prometheus 데이터소스를 만듭니다

Grafana는 실행 시 프로비저닝 파일을 읽어 데이터소스를 자동 등록합니다.

**파일: `monitoring/grafana/provisioning/datasources/prometheus.yml`**

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    uid: subject13-prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

### 실습 16. Grafana 대시보드 자동 불러오기 설정을 만듭니다

**파일: `monitoring/grafana/provisioning/dashboards/dashboard.yml`**

```yaml
apiVersion: 1
providers:
  - name: '교과13 모니터링'
    orgId: 1
    folder: '교과13'
    type: file
    disableDeletion: true
    editable: false
    options:
      path: /var/lib/grafana/dashboards
```

### 실습 17. Grafana 대시보드 JSON을 작성합니다

이 파일은 이미 존재하는 지표만을 조회합니다. **임의의 성능 수치나 GPU 사용률 그래프를 추가하지 않습니다.** 파일 내용이 길지만 전체를 그대로 저장합니다.

**파일: `monitoring/grafana/dashboards/subject13_day10.json`**

```json
{
  "uid": "subject13-day10",
  "title": "교과 13 | FastAPI · Jetson 모니터링",
  "schemaVersion": 39,
  "editable": false,
  "refresh": "10s",
  "time": {
    "from": "now-15m",
    "to": "now"
  },
  "timezone": "browser",
  "tags": [
    "subject13",
    "day10"
  ],
  "panels": [
    {
      "id": 1,
      "title": "FastAPI 요청/초",
      "type": "timeseries",
      "gridPos": {
        "x": 0,
        "y": 0,
        "w": 8,
        "h": 7
      },
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "expr": "sum(rate(platform_http_requests_total{job=\"platform_api\"}[1m]))",
          "refId": "A",
          "legendFormat": "{{job}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "req/s"
        },
        "overrides": []
      },
      "options": {}
    },
    {
      "id": 2,
      "title": "HTTP 5xx 요청/초",
      "type": "timeseries",
      "gridPos": {
        "x": 8,
        "y": 0,
        "w": 8,
        "h": 7
      },
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "expr": "sum(rate(platform_http_requests_total{job=\"platform_api\",status=~\"5..\"}[1m]))",
          "refId": "A",
          "legendFormat": "{{job}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "req/s"
        },
        "overrides": []
      },
      "options": {}
    },
    {
      "id": 3,
      "title": "FastAPI p95 응답시간",
      "type": "timeseries",
      "gridPos": {
        "x": 16,
        "y": 0,
        "w": 8,
        "h": 7
      },
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum by (le) (rate(platform_http_request_duration_seconds_bucket{job=\"platform_api\"}[5m])))",
          "refId": "A",
          "legendFormat": "{{job}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "s"
        },
        "overrides": []
      },
      "options": {}
    },
    {
      "id": 4,
      "title": "FastAPI 프로세스 메모리",
      "type": "timeseries",
      "gridPos": {
        "x": 0,
        "y": 7,
        "w": 8,
        "h": 7
      },
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "expr": "process_resident_memory_bytes{job=\"platform_api\"}",
          "refId": "A",
          "legendFormat": "{{job}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "bytes"
        },
        "overrides": []
      },
      "options": {}
    },
    {
      "id": 5,
      "title": "Jetson 연결 상태(1=정상)",
      "type": "stat",
      "gridPos": {
        "x": 8,
        "y": 7,
        "w": 8,
        "h": 7
      },
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "expr": "platform_jetson_up{job=\"platform_api\"}",
          "refId": "A",
          "legendFormat": "{{job}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "short"
        },
        "overrides": []
      },
      "options": {}
    },
    {
      "id": 6,
      "title": "카메라 최신 프레임(1=정상)",
      "type": "stat",
      "gridPos": {
        "x": 16,
        "y": 7,
        "w": 8,
        "h": 7
      },
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "expr": "platform_jetson_camera_ready{job=\"platform_api\"}",
          "refId": "A",
          "legendFormat": "{{job}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "short"
        },
        "overrides": []
      },
      "options": {}
    },
    {
      "id": 7,
      "title": "Jetson 실측 처리 FPS",
      "type": "timeseries",
      "gridPos": {
        "x": 0,
        "y": 14,
        "w": 8,
        "h": 7
      },
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "expr": "platform_jetson_processing_fps{job=\"platform_api\"}",
          "refId": "A",
          "legendFormat": "{{job}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "fps"
        },
        "overrides": []
      },
      "options": {}
    },
    {
      "id": 8,
      "title": "TensorRT 최근 p95 추론시간",
      "type": "timeseries",
      "gridPos": {
        "x": 8,
        "y": 14,
        "w": 8,
        "h": 7
      },
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "expr": "platform_jetson_inference_recent_p95_ms{job=\"platform_api\"}",
          "refId": "A",
          "legendFormat": "{{job}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "ms"
        },
        "overrides": []
      },
      "options": {}
    },
    {
      "id": 9,
      "title": "Jetson 오류 누적 (재시작 시 초기화)",
      "type": "timeseries",
      "gridPos": {
        "x": 16,
        "y": 14,
        "w": 8,
        "h": 7
      },
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "expr": "platform_jetson_errors_seen{job=\"platform_api\"}",
          "refId": "A",
          "legendFormat": "{{job}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "short"
        },
        "overrides": []
      },
      "options": {}
    },
    {
      "id": 10,
      "title": "FastAPI CPU 사용률(1 CPU = 100%)",
      "type": "timeseries",
      "gridPos": {
        "x": 0,
        "y": 21,
        "w": 8,
        "h": 7
      },
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "expr": "rate(process_cpu_seconds_total{job=\"platform_api\"}[1m])*100",
          "refId": "A",
          "legendFormat": "{{job}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent"
        },
        "overrides": []
      },
      "options": {}
    },
    {
      "id": 11,
      "title": "Jetson CUDA 사용 가능(사용률 아님)",
      "type": "stat",
      "gridPos": {
        "x": 8,
        "y": 21,
        "w": 8,
        "h": 7
      },
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "expr": "platform_jetson_gpu_available{job=\"platform_api\"}",
          "refId": "A",
          "legendFormat": "{{job}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "short"
        },
        "overrides": []
      },
      "options": {}
    },
    {
      "id": 12,
      "title": "Jetson 누적 프레임 수",
      "type": "timeseries",
      "gridPos": {
        "x": 16,
        "y": 21,
        "w": 8,
        "h": 7
      },
      "datasource": {
        "type": "prometheus",
        "uid": "subject13-prometheus"
      },
      "targets": [
        {
          "expr": "platform_jetson_frames_seen{job=\"platform_api\"}",
          "refId": "A",
          "legendFormat": "{{job}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "short"
        },
        "overrides": []
      },
      "options": {}
    }
  ],
  "annotations": {
    "list": []
  },
  "version": 1
}
```

**대시보드 확인 포인트:** API 요청/초, 응답시간 지표, Jetson 연결·카메라 준비 상태, 처리 FPS, 최근 추론시간, 오류 누적값을 구분합니다. 측정 전에 FPS가 0으로 보일 수 있으며 이는 성능이 0이라는 확정 판단이 아닙니다.

### 실습 18. 기존 PC Compose에 모니터링 서비스를 추가합니다

기존 `compose.yaml`을 통째로 덮어쓰지 않습니다. 새 **추가 설정 파일**을 만들고 `-f`로 함께 실행합니다.

**의사코드**

```text
기존 compose.yaml
  ├─ api
  ├─ db
  ├─ redis
  └─ worker
      +
compose.monitoring.yaml
  ├─ Prometheus
  └─ Grafana
      ↓
같은 platform Docker 네트워크에서 API 지표 수집
```

**파일: `compose.monitoring.yaml`**

```yaml
# 10일차: 기존 compose.yaml에 덧씌우는 모니터링 구성입니다.
# 항상 -f compose.yaml -f compose.monitoring.yaml 순서로 사용합니다.
services:
  api:
    environment:
      DAY08_JETSON_URL: ${DAY08_JETSON_URL:-}
      DAY08_JETSON_API_KEY: ${DAY08_JETSON_API_KEY:-}

  prometheus:
    image: prom/prometheus:v3.2.1
    restart: unless-stopped
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=3d
      - --web.enable-lifecycle
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./monitoring/alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
      - monitoring_prometheus_data:/prometheus
    depends_on:
      api: {condition: service_healthy}
    ports:
      - "${MONITORING_BIND_HOST:-127.0.0.1}:9090:9090"
    networks: [platform]

  grafana:
    image: grafana/grafana:11.6.0
    restart: unless-stopped
    depends_on: [prometheus]
    environment:
      GF_SECURITY_ADMIN_USER: ${GRAFANA_ADMIN_USER:? .env에 Grafana 관리 계정을 입력하세요}
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD:? .env에 Grafana 관리 비밀번호를 입력하세요}
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - monitoring_grafana_data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards:ro
    ports:
      - "${MONITORING_BIND_HOST:-127.0.0.1}:3000:3000"
    networks: [platform]

volumes:
  monitoring_prometheus_data:
  monitoring_grafana_data:
```

`monitoring_prometheus_data`와 `monitoring_grafana_data`는 별도 Docker 볼륨입니다. 기존 `pg_data`·`redis_data`를 삭제하지 않습니다. `MONITORING_BIND_HOST=127.0.0.1`이 기본값이므로 브라우저를 실행한 PC 내부에서만 접속할 수 있습니다.

### 실습 19. 환경변수를 추가합니다

기존 `.env` 파일을 **유지한 상태로 맨 아래에** 다음 항목을 추가합니다. 학생별 Jetson IP와 암호를 사용합니다. 이 부분은 템플릿으로만 제공하며 **실제 인증키를 README·Git에 기록하지 않습니다.**

```dotenv
# 10일차 PC Docker 설정 예시 — 본인 환경에 맞게 수정
DAY08_JETSON_URL=http://192.168.0.50:8018
DAY08_JETSON_API_KEY=CHANGE_TO_EXISTING_LONG_SECRET_KEY
MONITORING_BIND_HOST=127.0.0.1
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=CHANGE_TO_STRONG_UNIQUE_PASSWORD
```

`DAY08_JETSON_API_KEY`는 **8~9일차 Jetson과 PC에서 사용한 같은 실제 키**여야 합니다. `CHANGE_TO...` 문자열 그대로 사용하지 않습니다. Jetson이 없는 실습 단계에서는 URL과 키를 비워 놓아도 PC API 지표를 확인할 수 있지만 `platform_jetson_up=0`이 나타납니다.

```bash
# .env가 Git에 들어가지 않는지 확인
git check-ignore .env
```

`.env`가 무시되지 않으면 `.gitignore`에 `.env`, `.env.*`, `!.env.example`을 추가합니다. 키가 이미 커밋되어 있다면 단순 `.gitignore` 추가로 해결되지 않으므로 해당 키를 폐기·재발급해야 합니다.

### 실습 20. 모니터링 화면 HTML을 추가합니다

**파일: `web/monitoring.html`**

```html
<!doctype html>
<html lang="ko">
<head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>교과13 10일차 모니터링</title>
<style>body{font-family:system-ui;max-width:850px;margin:40px auto;padding:0 15px;color:#263447}section{padding:18px;border:1px solid #dae1e9;border-radius:12px;margin:16px 0}a{color:#1766a7}code{background:#eef3f8;padding:3px 5px}.hint{color:#637083}</style></head>
<body><h1>교과 13 · 10일차 모니터링</h1>
<p>9일차 Jetson TensorRT 서비스와 PC FastAPI의 실제 실행 상태를 확인합니다.</p>
<section><h2>1. 모니터링 화면 열기</h2>
<p>아래 두 서비스는 10일차 Docker Compose가 시작되어 있어야 접속됩니다.</p>
<p><a href="http://127.0.0.1:3000" target="_blank" rel="noopener">Grafana (127.0.0.1:3000)</a> · <a href="http://127.0.0.1:9090/targets" target="_blank" rel="noopener">Prometheus Targets (127.0.0.1:9090)</a></p>
<p class="hint">원격 PC에서는 브라우저를 실행한 컴퓨터의 localhost가 아닌 수업 PC의 접근 가능한 주소를 사용해야 합니다.</p></section>
<section><h2>2. 원본 지표 확인</h2><p><a href="/metrics">Prometheus 지표(/metrics)</a> · <a href="/health">FastAPI 상태(/health)</a> · <a href="/jetson-stream">9일차 실시간 화면(/jetson-stream)</a></p></section>
<section><h2>3. 해석 기준</h2><p><code>platform_jetson_up=0</code>이면 Jetson 텔레메트리 접근에 실패했습니다. <code>platform_jetson_camera_ready=0</code>이면 최신 카메라 프레임이 없습니다.</p><p>이 값들은 실제 지표이고 위험동작 판정이 아닙니다. GPU 사용률은 이 대시보드에서 제공하지 않습니다.</p></section></body></html>
```

이 페이지는 계측값을 스스로 계산하지 않고 Grafana·Prometheus와 기존 실시간 화면으로 안내합니다.

## PART 5. 실제 Docker와 Jetson에서 서비스를 실행합니다

### 실습 21. PC Compose 통합 설정의 문법을 확인합니다

먼저 두 Compose 파일을 합친 결과를 확인합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
docker compose -f compose.yaml -f compose.monitoring.yaml config --quiet
```

오류가 없다면 Compose 설정을 읽을 수 있습니다. `GRAFANA_ADMIN_PASSWORD` 등 필수 환경변수를 입력하지 않았다면 구성 확인 단계에서 오류가 날 수 있으므로 `.env`부터 확인합니다.

### 실습 22. PC의 FastAPI 이미지를 다시 빌드합니다

**중요:** Dockerfile이 소스코드를 이미지 안으로 복사하므로 `app/day10_*.py`·`web/monitoring.html`을 추가한 뒤 반드시 다시 빌드합니다.

```bash
docker compose -f compose.yaml -f compose.monitoring.yaml build api
docker compose -f compose.yaml -f compose.monitoring.yaml up -d db redis api worker prometheus grafana
docker compose -f compose.yaml -f compose.monitoring.yaml ps
```

`api`, `db`, `redis`, `worker`, `prometheus`, `grafana`가 예상대로 실행되는지 확인합니다. `api`가 준비되지 않았다면 먼저 다음 로그를 확인하세요.

```bash
docker compose -f compose.yaml -f compose.monitoring.yaml logs --tail=60 api
docker compose -f compose.yaml -f compose.monitoring.yaml logs --tail=60 prometheus
docker compose -f compose.yaml -f compose.monitoring.yaml logs --tail=60 grafana
```

### 실습 23. Jetson 새 이미지를 빌드하고 기존 추론 기능을 유지합니다

9일차와 같이 PC에서 수정한 `deploy/day08_jetson/` 코드를 Jetson 장치의 동일 경로로 동기화한 뒤 **Jetson에서** 실행합니다. 전송 방식은 8~9일차에 사용한 `scp` 또는 내부 저장소 방식을 그대로 사용합니다. 엔진·비밀키·`.env.jetson`은 덮어쓰지 않습니다.

**Jetson 터미널:**

```bash
cd ~/subject13_ai_platform/deploy/day08_jetson
# 이전 일차에 사용한 실제 Jetson 경로로 이동하세요.
docker compose -f compose.jetson.yaml config --quiet
docker compose -f compose.jetson.yaml build
docker compose -f compose.jetson.yaml up -d
docker compose -f compose.jetson.yaml ps
docker compose -f compose.jetson.yaml logs --tail=80
```

> 기존 9일차 Compose가 여러 GPU 컨테이너를 동시에 실행하도록 설정되어 있다면 메모리가 부족할 수 있습니다. 전날과 동일하게 **한 개의 실제 TensorRT Pose 추론 컨테이너**가 실행되는지 확인합니다.

### 실습 24. Jetson Telemetry 인증 API를 직접 확인합니다

**Jetson 장치 터미널**에서 다음을 실행합니다. 환경변수 `JETSON_API_KEY`가 이미 로드되어 있지 않다면 직접 입력할 때 셸 기록에 남지 않도록 주의합니다.

```bash
# 예: 본인 장치의 실제 파일과 키 설정을 확인한 뒤 실행
curl -i http://127.0.0.1:8018/api/v1/jetson/telemetry
```

인증 헤더가 없으면 **HTTP 401**이어야 합니다. 기존 Jetson 키를 넣어 정상 호출하는 예시는 다음과 같습니다.

```bash
read -rsp 'Jetson API key: ' JETSON_API_KEY; echo
curl -s -H "X-Jetson-Key: $JETSON_API_KEY"   http://127.0.0.1:8018/api/v1/jetson/telemetry | python3 -m json.tool
unset JETSON_API_KEY
```

9일차 카메라가 정상 실행 중이라면 `camera_ready`, `measured_fps`, `frames_total`, `inference_last_ms`, `inference_p95_ms`, `error_count` 같은 값이 나타납니다. 지표에는 원본 이미지나 비밀키가 없어야 합니다.

### 실습 25. PC FastAPI `/metrics`에서 실제 지표를 확인합니다

**PC 터미널:**

```bash
curl -s http://127.0.0.1:8000/metrics | grep -E '^platform_(http_requests_total|jetson_up|jetson_camera_ready|jetson_processing_fps|jetson_inference_last_ms)' | head -25
```

처음에는 특정 API 호출 지표가 없을 수 있습니다. 새 터미널에서 몇 차례 `/health`를 호출한 뒤 다시 확인합니다.

```bash
for i in 1 2 3 4 5; do curl -s -o /dev/null http://127.0.0.1:8000/health; done
curl -s http://127.0.0.1:8000/metrics | grep '^platform_http_requests_total'
```

정상적으로 기록되면 `method`, `route`, `status`의 조합별 누적 요청 수가 증가합니다. 테스트 중 일부 요청이 HTTP 404가 되어도 **오류 상태를 지표로 기록하는 것이 정상**입니다.

### 실습 26. Prometheus 수집 대상이 정상인지 확인합니다

PC 브라우저에서 엽니다.

```text
http://127.0.0.1:9090/targets
```

`platform_api`와 `prometheus` Target이 `UP`이면 수집 경로가 정상입니다. `platform_api=DOWN`이면 Prometheus 컨테이너에서 `api:8000`에 접근할 수 있는지 확인합니다. **Target UP은 Jetson 카메라 정상과 동일한 뜻이 아닙니다.** Jetson은 `platform_jetson_up`을 별도로 확인합니다.

Prometheus의 Query 창에서 다음 PromQL을 각각 실행합니다.

```promql
up{job="platform_api"}
platform_jetson_up{job="platform_api"}
platform_jetson_camera_ready{job="platform_api"}
platform_jetson_processing_fps{job="platform_api"}
sum(rate(platform_http_requests_total{job="platform_api"}[1m]))
```

지표가 보이지 않으면 먼저 10~30초 기다린 후 `/metrics` 응답·Prometheus Target 상태·FastAPI 로그 순서로 확인합니다.

### 실습 27. Grafana에서 자동 등록된 대시보드를 확인합니다

브라우저에 다음 주소를 입력합니다.

```text
http://127.0.0.1:3000
```

`.env`에 설정한 Grafana 관리자 계정으로 로그인합니다. 왼쪽 메뉴의 대시보드에서 **교과13** 폴더 → **교과 13 | FastAPI · Jetson 모니터링**을 엽니다.

오른쪽 상단의 시간 범위를 **Last 15 minutes**로 설정합니다. API 요청 그래프·FPS·카메라 준비 상태·추론시간 등을 확인합니다. 그래프의 수치는 실제 시스템 상태와 시간 경과에 따라 달라집니다.

### 실습 28. 기존 9일차 카메라 화면과 DB 저장 기능이 유지되는지 확인합니다

브라우저에서 **기존 화면을 다시 엽니다.**

```text
http://127.0.0.1:8000/jetson-stream
http://127.0.0.1:8000/api/v1/summary
http://127.0.0.1:8000/monitoring
```

9일차 실시간 화면의 최신 프레임이 갱신되는지 확인하고, 필요할 때 **기존 '저장' 기능으로 새 프레임 한 건을 PostgreSQL에 기록**합니다. `/api/v1/summary` 또는 기존 이벤트 조회 API에서 해당 기록을 확인합니다. **모니터링 지표 수집 자체는 새로운 추론 이벤트를 DB에 저장하지 않습니다.**

## PART 6. 장애 상황에서 지표 변화와 복구를 확인합니다

### 실습 29. Jetson 연결 중단 시 `platform_jetson_up`을 확인합니다

Jetson이 실행 중인 상태에서 **장치 전원이나 케이블을 무리하게 뽑지 않고**, 9일차 TensorRT Pose 서비스를 일시 중지합니다. Jetson 프로젝트 폴더에서 실제 서비스명을 먼저 확인하세요.

```bash
# Jetson 터미널
docker compose -f compose.jetson.yaml ps
# 다음 명령의 SERVICE_NAME은 위 결과의 실제 Compose 서비스 이름으로 변경
# docker compose -f compose.jetson.yaml stop SERVICE_NAME
```

이때 PC에서 `/metrics`를 조회하면 `platform_jetson_up=0`이 되어야 합니다. Prometheus는 기본 10초 간격으로 수집하므로 Grafana 반영에는 약간의 시간이 걸립니다. 해당 상태가 1분간 지속되면 **JetsonTelemetryUnavailable** 경고가 활성화될 수 있습니다.

Jetson 서비스를 원래대로 시작합니다.

```bash
# docker compose -f compose.jetson.yaml start SERVICE_NAME
```

새 프레임을 얻은 후 `platform_jetson_up=1`, `platform_jetson_camera_ready=1`로 회복되는지 확인합니다. 복구 전에 저장된 예전 프레임을 최신이라고 판단해서는 안 됩니다.

### 실습 30. 카메라만 중단되었을 때와 Jetson 전체 중단을 구분합니다

카메라 앱의 영상 송출을 **정상적인 종료 버튼**으로 잠시 멈춥니다. Jetson 서버는 실행한 상태로 둡니다.

```text
예상되는 상태 조합:
Jetson API 인증 연결 성공 → platform_jetson_up = 1
최근 3초 새 프레임 없음 → platform_jetson_camera_ready = 0
현재 처리 FPS → platform_jetson_processing_fps = 0
```

카메라 앱 송출을 다시 시작한 뒤 최신 프레임이 정상인지 확인합니다. 네트워크 상태나 Jetson 재연결 처리에 따라 실제 회복 시간은 달라질 수 있습니다. `JetsonCameraUnavailable` 경고는 이 상태가 **1분 이상** 유지되면 활성화되도록 설정했습니다.

### 실습 31. Prometheus Alert 조건을 확인합니다

```text
http://127.0.0.1:9090/alerts
```

`inactive` → `pending` → `firing` 상태를 구분해서 확인합니다. 오늘은 메일·Slack·문자 등 외부 알림 전송을 구현하지 않습니다. 장애 검증을 마친 후 **Jetson 서비스와 카메라를 반드시 원래 상태로 복구**합니다.

### 실습 32. 자주 발생하는 오류를 점검합니다

| 증상 | 확인할 원인 | 해결 방법 |
|---|---|---|
| `ModuleNotFoundError: prometheus_client` | Python 패키지 미설치·Docker 재빌드 누락 | `requirements.docker.txt` 추가 후 `build api` 수행 |
| `GET /metrics` 404 | `app/main.py`의 라우터 연결 누락 | `app.include_router(day10_router)` 확인 |
| `/metrics` 500 | Jetson 요청 예외·코드 중복 설치 | API 로그 및 계측 함수 확인 |
| Prometheus Target `DOWN` | Docker 네트워크·포트·API 비정상 | `docker compose ... ps`, `logs api prometheus` 확인 |
| `platform_jetson_up=0` | Jetson URL·포트·인증·장치 중단 | `.env` 두 변수와 인증 API 직접 확인 |
| `camera_ready=0` | 최신 프레임 없음·영상 입력 장애 | 9일차 카메라 설정·Jetson 로그·네트워크 확인 |
| Grafana 빈 화면 | 데이터소스·프로비저닝 위치 오류 | Grafana 로그·대시보드 JSON 경로 확인 |
| 포트 3000/9090 충돌 | 다른 개발 서버가 포트 점유 | 충돌 서비스 정리 후 재실행 |
| Docker Compose `config` 오류 | 필수 `.env` 변수 누락 | 기존 7일차 변수와 10일차 관리자 계정 확인 |
| TensorRT 엔진 오류 | 장치 변경·9일차 엔진/Manifest 문제 | 9일차 Jetson에서 엔진을 다시 검증 |

**오류 해결 순서:** `기존 9일차 추론 → Jetson telemetry → PC /metrics → Prometheus Targets → Grafana` 순서로 문제 범위를 좁힙니다. 기존 PostgreSQL 볼륨은 삭제하지 않습니다.

## PART 7. 자동 테스트·운영 문서·Git 정리

### 실습 33. 10일차 자동 테스트 코드를 작성합니다

오늘의 테스트는 실제 Jetson GPU가 없어도 API 계측과 설정이 잘못되지 않았는지 확인하는 **단위·계약 테스트**를 포함합니다. Jetson 실측 자체를 대체하지는 않습니다.

아래 파일을 지정된 위치에 작성합니다. 8·9일차 테스트 파일은 그대로 보존합니다.

**파일: `tests/test_day10_telemetry.py`**

```python
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parents[1] / 'deploy' / 'day08_jetson'))
from day10_telemetry import PoseTelemetry


def test_empty():
    assert PoseTelemetry().snapshot() == {
        'inference_count': 0, 'inference_last_ms': None,
        'inference_p95_ms': None, 'error_count': 0,
    }


def test_counts_and_p95():
    tracker = PoseTelemetry()
    for i in range(1, 101):
        tracker.record_inference(float(i))
    snap = tracker.snapshot()
    assert snap['inference_count'] == 100
    assert snap['inference_last_ms'] == 100.0
    assert snap['inference_p95_ms'] == 95.0
    tracker.record_error()
    assert tracker.snapshot()['error_count'] == 1


def test_last_two_hundred_samples():
    tracker = PoseTelemetry()
    for i in range(300):
        tracker.record_inference(float(i))
    snap = tracker.snapshot()
    assert snap['inference_count'] == 300
    assert snap['inference_last_ms'] == 299.0
    assert snap['inference_p95_ms'] == 289.0


def test_negative_ms_rejected():
    tracker = PoseTelemetry()
    try:
        tracker.record_inference(-1.0)
    except ValueError:
        pass
    else:
        raise AssertionError('negative time accepted')
```

**파일: `tests/test_day10_monitoring.py`**

```python
"""10일차: GPU·실제 Docker 없이 수행하는 Prometheus 계측 및 Jetson 계약 테스트."""
from fastapi import FastAPI, HTTPException
from fastapi.testclient import TestClient
from prometheus_client import generate_latest

from app.day10_api import router
from app import day10_metrics as m


def build_client():
    app = FastAPI()

    @app.get('/check/{item_id}')
    def check(item_id: str):
        return {"item": item_id}

    @app.get('/broken')
    def broken():
        raise HTTPException(503, 'service offline')

    app.include_router(router)
    m.install_metrics(app)
    return TestClient(app)


def test_metrics_endpoint_contains_names(monkeypatch):
    monkeypatch.delenv('DAY08_JETSON_URL', raising=False)
    monkeypatch.delenv('DAY08_JETSON_API_KEY', raising=False)
    client = build_client()
    response = client.get('/metrics')
    assert response.status_code == 200
    assert 'platform_jetson_up' in response.text
    assert 'platform_http_requests_total' in response.text
    assert 'text/plain' in response.headers['content-type']


def test_http_metrics_no_dynamic_user_id_label():
    client = build_client()
    assert client.get('/check/a-user-specific-777').status_code == 200
    data = client.get('/metrics').text
    assert 'route="/check/{item_id}"' in data
    assert 'a-user-specific-777' not in data


def test_http_5xx_counted():
    client = build_client()
    assert client.get('/broken').status_code == 503
    data = client.get('/metrics').text
    assert 'status="503"' in data


def test_prometheus_scrape_not_observed():
    client = build_client()
    client.get('/metrics')
    data = generate_latest().decode()
    assert 'route="/metrics"' not in data


def test_jetson_unconfigured_sets_not_ready(monkeypatch):
    monkeypatch.delenv('DAY08_JETSON_URL', raising=False)
    monkeypatch.delenv('DAY08_JETSON_API_KEY', raising=False)
    assert m.poll_jetson_telemetry() is False
    assert m.JETSON_UP._value.get() == 0


def test_jetson_valid_telemetry(monkeypatch):
    monkeypatch.setenv('DAY08_JETSON_URL', 'http://192.0.2.10:8018')
    monkeypatch.setenv('DAY08_JETSON_API_KEY', '123456789012345678901234')
    monkeypatch.setattr(m, 'request_jetson_telemetry', lambda url, key: {
        'camera_ready': True, 'gpu_available': True, 'measured_fps': 7.1,
        'frames_total': 123, 'error_count': 2, 'inference_last_ms': 32.4,
        'inference_p95_ms': 39.1,
    })
    assert m.poll_jetson_telemetry() is True
    assert m.JETSON_UP._value.get() == 1
    assert m.JETSON_FPS._value.get() == 7.1
    assert m.JETSON_INFERENCE_P95_MS._value.get() == 39.1


def test_jetson_unauthorized_or_network_error(monkeypatch):
    monkeypatch.setenv('DAY08_JETSON_URL', 'http://192.0.2.10:8018')
    monkeypatch.setenv('DAY08_JETSON_API_KEY', '123456789012345678901234')
    monkeypatch.setattr(m, 'request_jetson_telemetry', lambda url, key: (_ for _ in ()).throw(ValueError('unauthorized')))
    assert m.poll_jetson_telemetry() is False
    assert m.JETSON_UP._value.get() == 0
    assert m.JETSON_CAMERA_READY._value.get() == 0


def test_jetson_rejects_impossible_fps(monkeypatch):
    monkeypatch.setenv('DAY08_JETSON_URL', 'http://192.0.2.10:8018')
    monkeypatch.setenv('DAY08_JETSON_API_KEY', '123456789012345678901234')
    monkeypatch.setattr(m, 'request_jetson_telemetry', lambda url, key: {
        'camera_ready': True, 'gpu_available': True, 'measured_fps': -1,
        'frames_total': 10, 'error_count': 0,
    })
    assert m.poll_jetson_telemetry() is False
    assert m.JETSON_UP._value.get() == 0


def test_monitoring_page():
    client = build_client()
    assert client.get('/monitoring').status_code == 200
    assert '10일차 모니터링' in client.get('/monitoring').text
```

**파일: `tests/test_day10_config.py`**

```python
from pathlib import Path
import json
import yaml

ROOT = Path(__file__).resolve().parents[1]


def test_overlay_adds_only_monitoring():
    base = yaml.safe_load((ROOT / 'compose.yaml').read_text())
    overlay = yaml.safe_load((ROOT / 'compose.monitoring.yaml').read_text())
    assert all(x in base['services'] for x in ('db','redis','api','worker'))
    assert all(x in overlay['services'] for x in ('prometheus','grafana'))
    assert 'db' not in overlay['services']
    assert 'redis' not in overlay['services']
    assert 'worker' not in overlay['services']
    assert 'pg_data' not in overlay['volumes']


def test_prometheus_target_and_rule_names():
    conf = yaml.safe_load((ROOT / 'monitoring/prometheus.yml').read_text())
    jobs = {x['job_name']:x for x in conf['scrape_configs']}
    assert jobs['platform_api']['static_configs'][0]['targets'] == ['api:8000']
    rules = yaml.safe_load((ROOT / 'monitoring/alert_rules.yml').read_text())
    names = [r['alert'] for group in rules['groups'] for r in group['rules']]
    assert 'PlatformApiUnavailable' in names
    assert 'JetsonCameraUnavailable' in names


def test_dashboard_12_panels():
    path = ROOT / 'monitoring/grafana/dashboards/subject13_day10.json'
    dash = json.loads(path.read_text(encoding='utf-8'))
    assert len(dash['panels']) == 12
    assert all(x['datasource']['uid'] == 'subject13-prometheus' for x in dash['panels'])
    assert not any('gpu_utilization' in x['targets'][0]['expr'] for x in dash['panels'])
```

**파일: `tests/test_day10_jetson_route.py`**

```python
"""실제 YOLO 엔진 없이 Jetson 텔레메트리 API의 인증 및 상태 규격을 검증합니다."""
import importlib.util
import sys
import types
from pathlib import Path
from time import monotonic

from fastapi.testclient import TestClient

ROOT = Path(__file__).resolve().parents[1]
JETSON = ROOT / 'deploy/day08_jetson'


def load_jetson_module(monkeypatch):
    sys.path.insert(0, str(JETSON))
    monkeypatch.setitem(sys.modules, 'ultralytics', types.SimpleNamespace(YOLO=object))
    spec = importlib.util.spec_from_file_location('day10_test_jetson_app', JETSON / 'app_day09.py')
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    return module


def test_telemetry_api_requires_key(monkeypatch):
    module = load_jetson_module(monkeypatch)
    monkeypatch.setenv('JETSON_API_KEY', 'a' * 24)
    client = TestClient(module.app)
    assert client.get('/api/v1/jetson/telemetry').status_code == 401
    assert client.get('/api/v1/jetson/telemetry', headers={'X-Jetson-Key': 'bad'}).status_code == 401


def test_telemetry_api_has_actual_camera_data(monkeypatch):
    module = load_jetson_module(monkeypatch)
    monkeypatch.setenv('JETSON_API_KEY', 'a' * 24)
    monkeypatch.setattr(module.torch.cuda, 'is_available', lambda: True)
    module.POSE_TELEMETRY.record_inference(24.7)
    module.STATE.update(camera_status='running', measured_fps=7.4, frames_total=51,
                        latest={'created_monotonic': monotonic()})
    client = TestClient(module.app)
    result = client.get('/api/v1/jetson/telemetry', headers={'X-Jetson-Key': 'a' * 24})
    assert result.status_code == 200, result.text
    data = result.json()
    assert data['camera_ready'] is True
    assert data['inference_last_ms'] == 24.7
    assert data['gpu_available'] is True
    assert 'jpeg_base64' not in data
    assert 'JETSON_API_KEY' not in data
```

**테스트 코드를 이해합니다.**

- FastAPI의 요청 계측과 `/metrics` 반환 형식을 확인합니다.
- Jetson의 상태 API가 비밀키 없이 열리지 않는지 검사합니다.
- 카메라 프레임이 오래되면 `camera_ready=False`로 바뀌는지 확인합니다.
- Docker Compose·Prometheus·Grafana 설정에 필요한 서비스와 파일 경로가 정의되어 있는지 검사합니다.

**PC 가상환경에서 실행:**

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m pytest -q tests/test_day10_telemetry.py tests/test_day10_monitoring.py   tests/test_day10_config.py tests/test_day10_jetson_route.py
```

테스트가 사용하는 패키지가 부족하면 7일차 `requirements.docker.txt`에 기록된 패키지를 설치해야 합니다. 실제 PostgreSQL을 연결하지 않는 모의 테스트도 포함되어 있으며, **자동 테스트 통과만으로 Jetson GPU·Docker Compose 운영 검증이 완료된 것은 아닙니다.**

기존 8·9일차 테스트도 환경이 준비되었다면 별도로 실행하여 변경 사항이 기존 API를 깨지 않았는지 확인합니다.

```bash
python -m pytest -q tests/test_day08.py tests/test_day09.py
```

### 실습 34. 운영 점검 결과를 문서로 정리합니다

`docs/day10_monitoring_runbook.md`를 새로 생성하고 다음 내용을 작성합니다. 꺾쇠괄호로 표시된 자리는 **학생이 직접 측정한 값**으로 바꿉니다.

````markdown
# 교과 13 · 10일차 서비스 모니터링 운영 기록

## 1. 사용 환경
- PC Docker Compose 실행 결과: [정상/오류]
- Jetson Orin Nano Super 및 JetPack: [직접 확인]
- Jetson Pose 모델·실행 방식: YOLO11n-Pose / TensorRT FP16
- 카메라 입력: [IP Webcam/USB 카메라]

## 2. 상태 확인
- `/health` 응답: [실제 값]
- Prometheus `platform_api` Target: [UP/DOWN]
- Grafana 대시보드 접속: [정상/오류]
- `platform_jetson_up`: [실제 값]
- `platform_jetson_camera_ready`: [실제 값]

## 3. 실제 측정
- Jetson 처리 FPS: [측정값]
- 마지막 추론시간(ms): [측정값 또는 미측정]
- 최근 p95 추론시간(ms): [측정값 또는 미측정]
- API 지연/오류: [관찰값]

## 4. 장애·복구 실습
- 발생한 문제: [직접 기록]
- Prometheus·Grafana 지표 변화: [직접 기록]
- 복구 명령·수행 내용: [직접 기록]
- 정상 복귀 확인: [직접 기록]

## 5. 보안 및 운영
- 인증 키·DB 비밀번호는 보고서에 적지 않습니다.
- 실제 추론·PostgreSQL 데이터와 모니터링 지표는 구분합니다.

## 6. 다음 실습
- 11일차에는 동일한 AI 플랫폼을 허용된 클라우드 GPU 서버로 확장합니다.
````

### 실습 35. README와 시스템 구조도를 수정합니다

기존 `README.md`에 다음 단락을 추가합니다.

````markdown
## 10일차 — Prometheus·Grafana 모니터링

- 기존 FastAPI와 Jetson 텔레메트리를 `/metrics`로 노출합니다.
- PC Compose에 Prometheus·Grafana를 추가했습니다.
- 기존 DB·Redis·Celery와 9일차 추론 화면은 그대로 사용합니다.
- PC 접속: `http://127.0.0.1:8000/monitoring`
- Prometheus: `http://127.0.0.1:9090/targets`
- Grafana: `http://127.0.0.1:3000`
- 운영 기록: `docs/day10_monitoring_runbook.md`
````

기존 `docs/architecture.md`에도 다음 구조를 추가합니다.

```text
9일차 Jetson TensorRT 카메라 → Jetson Telemetry(인증 API)
                                           ↓
7일차 PC FastAPI ── /metrics ──────── Prometheus ── Grafana
       │
       ├─ PostgreSQL (이벤트 저장)
       ├─ Redis + Celery (비동기 작업)
       └─ 9일차 Jetson 실시간 웹 화면
```

### 실습 36. Git 제외 규칙을 다시 확인합니다

`git status` 이전에 `.gitignore`를 확인합니다. 기존 교과의 모델·엔진·영상·인증키는 계속 제외되어야 합니다.

```gitignore
# 10일차 추가 확인
.env
.env.*
!.env.example
monitoring/data/
*.engine
*.pt
*.mp4
```

소스코드에 실제 비밀키가 있는지 다시 확인합니다. `.env`와 Git 관리대상에 올라가지 않아야 할 원본 데이터는 공개 저장소에 넣지 않습니다.

# 6. 오늘 작성한 내용의 최종 정리 및 GitHub 업로드

## 6.1. 오늘 완성한 기능

```text
기존 9일차 실시간 Jetson TensorRT Pose
              ↓ 실측 FPS·추론시간·오류
       Jetson Telemetry (인증)
              ↓
기존 PC FastAPI + 요청 지표 ── /metrics
              ↓
           Prometheus
              ↓
        Grafana 대시보드
              ↓
   서비스 오류·연결 장애·복구 확인
```

오늘은 AI 모델을 새로 학습하거나 TensorRT 엔진을 다시 만드는 날이 아닙니다. 이미 실행 중인 모델과 서비스를 **관찰하고 장애를 파악하는 운영 기능**을 구현했습니다. 저장된 AI 예측값과 운영 지표는 서로 다른 데이터이며, **11일차 Cloud 서버로 옮겨도 같은 방식으로 관찰할 수 있습니다.**

## 6.2. 최종 자가 체크리스트

- [ ] 기존 `subject13_ai_platform`을 계속 사용했습니다.
- [ ] 9일차 Jetson TensorRT Pose·카메라·PC PostgreSQL을 유지했습니다.
- [ ] `app/day10_metrics.py` 및 `app/day10_api.py`를 추가했습니다.
- [ ] 기존 `app/main.py`에 10일차 계측·라우터를 연결했습니다.
- [ ] Jetson Telemetry API는 올바른 `X-Jetson-Key`를 요구합니다.
- [ ] `/metrics`에 FastAPI 요청 수와 Jetson 상태 지표가 나타납니다.
- [ ] Prometheus `platform_api` Target이 `UP`입니다.
- [ ] Grafana에서 실측 지표를 확인했습니다.
- [ ] Jetson 중지와 카메라 중지의 지표 차이를 확인하고 복구했습니다.
- [ ] 9일차의 실시간 추론 화면과 DB 저장이 여전히 작동합니다.
- [ ] 자동 테스트와 운영 기록을 작성했습니다.
- [ ] 실제 비밀키·엔진·영상·`.env`는 Git에 추가하지 않았습니다.

## 6.3. Git 커밋을 남깁니다

기존 Git 저장소에서 변경 파일을 확인합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git status
```

10일차에 추가·수정한 소스와 설정만 선택하여 스테이징합니다.

```bash
git add app/day10_metrics.py app/day10_api.py app/main.py   deploy/day08_jetson/day10_telemetry.py deploy/day08_jetson/app_day09.py   deploy/day08_jetson/Dockerfile.jetson requirements.docker.txt   compose.monitoring.yaml monitoring/ web/monitoring.html   scripts/20_check_day10_inputs.py   tests/test_day10_telemetry.py tests/test_day10_monitoring.py   tests/test_day10_config.py tests/test_day10_jetson_route.py   docs/day10_monitoring_runbook.md docs/architecture.md README.md .gitignore

git status
```

`git status`에서 `.env`, TensorRT `.engine`, `yolo11n-pose.pt`, 비공개 영상, PostgreSQL 데이터 파일이 스테이징되지 않았는지 확인합니다.

```bash
git commit -m "교과13 10일차: FastAPI Jetson Prometheus Grafana 모니터링 구축"
git log -1 --oneline
git status
```

외부 업로드가 허용된 환경에서만 `git push origin main`을 실행합니다. 폐쇄망에서는 허가된 내부 Git 저장소를 사용합니다.

## 6.4. 다음 11일차와 연결합니다

11일차에는 이 프로젝트의 FastAPI·DB·비동기 처리 구조를 유지하면서 **AWS EC2 GPU 등 허용된 Cloud 환경**으로 배포 범위를 확장합니다. 오늘의 `/metrics`, Prometheus, Grafana 구조도 Cloud 환경에서 재사용하되, 인증·네트워크·비용을 먼저 점검합니다.

---

### 10일차 실습에서 확인한 것과 아직 남은 것

오늘 작성한 코드는 **PC·Jetson의 실제 상태를 기록하기 위한 코드**입니다. Docker 엔진 실행, Prometheus 이미지 다운로드, Jetson TensorRT 추론·FPS·카메라 장애 실측은 학생 각자의 장비에서 진행해야 합니다. 테스트 코드의 모의 데이터나 빈 계측값을 현장 실측 결과라고 기록하지 않습니다.

