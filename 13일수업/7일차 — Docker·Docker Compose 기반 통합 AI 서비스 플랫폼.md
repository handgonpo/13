## 1. 오늘의 목표

교과 13의 1일차에는 교과 9 OCR 검사와 교과 10 동작인식의 저장 결과를 공통 JSON으로 연결했습니다. 2일차에는 PostgreSQL에 결과를 저장했고, 3일차에는 Redis·Celery를 이용해 JSONL·Parquet·Gold 결과를 백그라운드에서 생성했습니다. 4일차에는 기존 OCR·Pose/Action 프로그램과 실제 입력을 연결했으며, 5일차에는 TinyCNN을 학습하고 ONNX로 변환했습니다. 6일차에는 Raspberry Pi에서 CPU 추론과 GPIO 제어를 수행하고 PC 플랫폼으로 결과를 전송했습니다.

지금까지는 PC의 Python 가상환경에서 FastAPI를 실행하고, PostgreSQL과 Redis는 운영체제에서 각각 실행했습니다. 오늘은 **이미 만든 프로그램을 지우거나 새로 개발하지 않고**, 서로 연관된 서비스를 Docker 컨테이너로 실행하는 방법을 배웁니다.

오늘 수업을 마치면 다음을 직접 할 수 있어야 합니다.

- 컨테이너·이미지·볼륨·네트워크·Docker Compose의 역할을 설명할 수 있습니다.
- 1~6일차의 기존 FastAPI Router와 데이터 구조를 그대로 유지할 수 있습니다.
- Python용 `Dockerfile`과 Docker 실행용 의존성 파일을 작성할 수 있습니다.
- `compose.yaml`로 FastAPI·PostgreSQL·Redis·Celery Worker **4개 서비스**를 실행할 수 있습니다.
- 기존 PostgreSQL 기록을 백업·복원하거나, 원본 JSON에서 새 DB에 다시 이관할 수 있습니다.
- 브라우저와 Swagger에서 이전 기록·DB 조회·비동기 Export·Pi 기록 API를 사용할 수 있습니다.
- Docker를 재시작해도 PostgreSQL 기록과 완료된 Export 파일이 남아 있는지 확인할 수 있습니다.
- 로그·연결 상태·실행 오류를 확인하고 Git에 오늘 작업을 기록할 수 있습니다.

**완성 기준:** `docker compose ps`에 네 서비스가 실행 중으로 표시되고, `/containers`에서 PostgreSQL·Redis·Celery Worker 연결을 확인하며, 3일차 Export 작업이 `SUCCESS`로 완료되어 실제 JSONL·Parquet·Gold 파일이 만들어져야 합니다. DB 기록은 컨테이너 재시작 후에도 유지되어야 합니다.

오늘은 **새 AI 모델을 학습하지 않고, Jetson이나 TensorRT를 실행하지 않습니다.** 4일차의 기존 OCR·Pose/Action 모델과 6일차 Raspberry Pi는 독립된 실행환경을 유지하며, 중앙 플랫폼을 Docker화하는 데 집중합니다.

---

## 2. 오늘 수업의 전체 흐름

![[Pasted image 20261018131051.png]]

```text
1~6일차 subject13_ai_platform 코드·결과 확인
                ↓
Docker / Docker Compose 설치·실행 확인
                ↓
기존 PostgreSQL 기록 백업
                ↓
Docker용 Python 패키지 목록 작성
                ↓
Dockerfile · .dockerignore 작성
                ↓
compose.yaml 작성
                ↓
┌───────────────────────────────────────────────┐
│      Docker Compose · 공통 내부 네트워크       │
│                                               │
│  [FastAPI :8000]  ───▶  [PostgreSQL :5432]     │
│        │                      ▲               │
│        └──▶ [Redis :6379]     │               │
│                    │          │               │
│                    ▼          │               │
│              [Celery Worker] ─┘               │
└───────────────────────────────────────────────┘
                ↓
기존 DB 복원 또는 1일차 JSON 재이관
                ↓
컨테이너 상태·웹 API·이전 화면 확인
                ↓
3일차 Celery Export 요청·SUCCESS 확인
                ↓
PostgreSQL·Export 파일 보존/재시작 테스트
                ↓
오류 확인·구조 문서·README·Git
                ↓
[8일차] Jetson Orin Nano Super GPU Docker 확장
```

이 흐름에서 Docker가 관리하는 대상은 **PC의 중앙 서비스**입니다. `subject9_machinevision`의 Tesseract 서버, `subject10_pose_action`의 YOLO Pose Bridge, Raspberry Pi의 ONNX·GPIO 프로그램은 기존 실행환경에서 따로 운영합니다. Docker 내부에서 필요한 경우 이 서비스들에 HTTP로 연결할 수 있지만, 오늘의 필수 검증은 FastAPI·DB·Redis·Worker 통합입니다.

---

## 3. 실습 전에 알아야 할 핵심

### 3.1. 1~6일차에서 만든 파일은 그대로 사용합니다

새 프로젝트를 만들지 않습니다. 다음 내용은 앞에서 직접 구현한 것입니다.

| 기존 일차 | 이미 만든 것 | 오늘 활용하는 방법 |
|---|---|---|
| 1일차 | `app/main.py`, `app/schemas.py`, `app/records.py`, `web/index.html` | 기존 AI 결과 조회 API 유지 |
| 2일차 | `app/db.py`, `app/models.py`, `app/db_api.py`, PostgreSQL 테이블 | 동일한 테이블·CRUD 사용 |
| 3일차 | `app/celery_app.py`, `app/worker_tasks.py`, `app/data_pipeline.py` | Redis·Celery Worker를 별도 컨테이너로 실행 |
| 4일차 | `app/live_api.py`, 교과 9·10의 외부 AI 서비스 | 기존 HTTP Bridge 주소만 구분하여 유지 |
| 5일차 | `app/edge_api.py`, `edge/runtime.py`, ONNX·설정 파일 | PC 컨테이너에서 CPU ONNX 미리보기 유지(모델 파일이 있을 때) |
| 6일차 | `app/edge_device_api.py`, Pi 추론 결과 DB 저장 | 기존 수신·조회 API를 컨테이너로 이동 |

1일차의 `handoff/records/`, 3일차의 `data/day03/exports/`, 5일차의 `artifacts/day05/`는 **Git에 올리지 않은 실제 데이터·모델 파일**이므로 Docker 이미지에 직접 포함시키지 않습니다. 오늘은 필요할 때 컨테이너가 해당 폴더를 읽을 수 있도록 **볼륨 마운트**로 연결합니다.

**앞선 실습이 일부 완료되지 않았다면:** 최소한 1~6일차 교재에서 작성한 `app/`, `edge/`, `web/`, `scripts/` 소스와 `.env` 설정이 필요합니다. 저장 기록을 재이관하는 학생은 1일차의 두 JSON, 즉 `handoff/records/inspection_history.json`과 `action_jobs.json`이 필요합니다. 5일차 ONNX 미리보기까지 확인하려면 `artifacts/day05/tinycnn.onnx`와 `training_config.json`도 필요합니다. 모델·JSON이 없는 경우 **실제 자료를 전달받아 배치**해야 하며, 가상의 검사 결과나 가짜 모델을 새로 만들지 않습니다.

### 3.2. 이미지와 컨테이너는 서로 다릅니다

```text
Dockerfile + 소스코드 + 패키지 목록
                 ↓ docker build
             Docker Image
                 ↓ docker compose up
             Container 실행
```

Docker **Image**는 실행에 필요한 파일을 묶은 결과입니다. Docker **Container**는 그 이미지를 실행한 상태입니다. 오늘 FastAPI와 Celery Worker는 같은 Python 이미지를 사용하지만, **실행 명령이 달라 별개의 컨테이너**가 됩니다.

- **FastAPI 컨테이너:** HTTP 요청을 받고 API와 웹 화면을 제공합니다.
- **Celery Worker 컨테이너:** Redis에서 작업을 받아 3일차 내보내기 코드를 실행합니다.
- **PostgreSQL 컨테이너:** AI 이벤트 데이터를 테이블에 저장합니다.
- **Redis 컨테이너:** Celery의 Broker와 작업 상태 저장소로 사용됩니다.

### 3.3. 컨테이너 안의 `localhost`는 PC의 `localhost`와 다릅니다

Docker Compose에서는 같은 내부 네트워크에 있는 서비스를 **서비스 이름**으로 찾습니다.

```text
FastAPI / Worker 컨테이너
       ├─ DB_HOST=db          → PostgreSQL 컨테이너
       └─ REDIS_URL=redis://redis:6379/0 → Redis 컨테이너
```

2~3일차 PC `.env`에서는 `DB_HOST=127.0.0.1`, `REDIS_URL=redis://127.0.0.1:6379/0`을 사용했습니다. 이 값은 **PC에서 직접 실행하는 Python 프로그램**에 맞는 주소입니다. 오늘 Compose 파일에는 컨테이너 전용 `DB_HOST=db`, `REDIS_URL=redis://redis:6379/0`을 **별도로 지정**합니다.

**기존 `.env`의 PC 접속정보를 모두 지우거나 `DB_HOST=db`로 바꾸지 마세요.** 컨테이너는 Compose에서 받은 환경변수를 우선 사용합니다. 나중에 PC에서 기존 `.venv`로 테스트할 때의 주소와 구분해야 합니다.

### 3.4. Docker 이미지와 실제 데이터는 분리합니다

```text
코드와 패키지           → Docker Image
DB 테이블·레코드        → PostgreSQL named volume
Redis 상태/AOF          → Redis named volume
JSONL/Parquet/Gold      → PC의 data/day03/exports 폴더
실제 OCR/Action JSON    → PC의 handoff/records 폴더(읽기 전용)
ONNX 모델·전처리 설정   → PC의 artifacts/day05 폴더(읽기 전용)
```

컨테이너를 중지하고 다시 실행해도 DB와 파일을 유지하려면 이런 구분이 필요합니다. **이미지 재빌드는 DB 데이터를 백업하는 방법이 아닙니다.** 반대로 `docker compose down -v`는 볼륨까지 삭제하므로 오늘 실습에서 사용하지 않습니다.

### 3.5. 이번에는 기존 PC PostgreSQL과 Docker PostgreSQL이 별개입니다

2일차에 WSL Ubuntu에서 설치한 PostgreSQL은 PC에서 실행됩니다. 오늘 Docker Compose의 PostgreSQL은 **새로운 독립 DB 서버**입니다. 같은 데이터베이스 이름을 썼더라도 두 서버의 데이터는 자동으로 공유되지 않습니다.

따라서 기존 실습 기록(2일차 수동 CRUD, 4일차 실시간 추론, 6일차 Pi 결과 포함)을 이어 사용하려면 **PC PostgreSQL을 `pg_dump`로 백업하고 Docker PostgreSQL에 복원**해야 합니다. 기존 서버의 기록이 없어 처음부터 시작하는 학생만 **1일차 JSON을 다시 이관**합니다. 두 방법 중 **자신의 상황에 맞는 하나만 선택**합니다.

### 3.6. 7일차에 Docker로 넣지 않는 기능도 있습니다

교과 9의 Tesseract/OCR과 교과 10의 YOLO Pose/Random Forest는 기존의 별도 Python 가상환경에서 실행했습니다. 오늘 Docker Python 이미지는 **중앙 FastAPI·Celery·ONNX Runtime CPU에 필요한 패키지만** 설치합니다. 대용량 PyTorch·YOLO 모델을 이미지에 한꺼번에 넣지 않습니다.

4일차의 `/live` 화면에서 실제 OCR·Pose 추론을 사용하려면 기존 AI 서버가 별도로 실행되어 있어야 하고, 컨테이너에서 해당 서버의 주소에 접근할 수 있어야 합니다. 호스트에서만 `127.0.0.1`로 열려 있는 서버는 컨테이너에서 직접 접근할 수 없습니다. 7일차 기본 Compose만 실행했을 때 `/live/health`에 `connected: false`가 나올 수 있으며, **이를 Docker 통합 플랫폼 자체의 실패로 해석하지 않습니다.** 외부 AI 서버 재연결은 학습실의 네트워크 접근 정책을 확인한 뒤 진행합니다.

### 3.7. 외부 공개와 인증은 주의합니다

- PostgreSQL과 Redis의 포트는 **PC 외부에 공개하지 않습니다**. 같은 Compose 네트워크에서만 접근합니다.
- FastAPI는 처음에는 `127.0.0.1:8000`만 공개합니다. **Pi에서 접근해야 할 때만** 허용된 내부망 인터페이스로 변경합니다.
- 6일차의 `DAY06_DEVICE_API_KEY`는 기존 키를 재사용하고, `.env`는 Docker 이미지·Git에서 제외합니다.
- 오늘 Docker Compose 예제는 **교육용 내부망**을 전제로 합니다. 공개 인터넷 운영에는 HTTPS·사용자 인증·권한·접근 제어·비밀관리·백업 정책이 별도로 필요합니다.

---

## 4. 오늘 사용할 기술 스택

| 기술 | 실습에서 맡는 역할 |
|---|---|
| Docker Engine / Docker Desktop | 컨테이너 실행 환경 |
| Docker Compose v2 | FastAPI·PostgreSQL·Redis·Worker를 함께 실행 |
| `Dockerfile` / `.dockerignore` | 공통 Python 이미지 생성·빌드 대상 관리 |
| Python 3.12 / Uvicorn / FastAPI | 기존 플랫폼 API 실행 |
| PostgreSQL 16 / SQLAlchemy / Psycopg | 2~6일차 AI 이벤트 저장·조회 |
| Redis 7 / Celery Worker | 3일차 비동기 Export 작업 처리 |
| Docker Bridge Network | 서비스 이름 `db`, `redis`, `api`, `worker`로 내부 통신 |
| Docker Named Volume | PostgreSQL·Redis 데이터 보존 |
| Bind Mount | 기존 JSON·ONNX 및 Export 산출물 공유 |
| ONNX Runtime CPU | 5일차 ONNX 미리보기(모델 파일이 준비된 경우) |
| `pg_dump` / `pg_restore` | 기존 PostgreSQL 기록 인계 |
| Pytest · Git | 새로운 연결 코드 확인과 일차별 버전 관리 |

> **실습 기준:** Windows 11 + WSL2 Ubuntu 또는 Ubuntu x86_64 PC에서 Docker Compose v2를 사용합니다. Raspberry Pi·Jetson ARM64 이미지 빌드는 오늘 하지 않습니다. Docker Desktop을 사용하는 Windows 환경에서는 WSL 통합이 켜져 있어야 합니다.

---

# 5. 처음부터 끝까지 따라 하는 상세 실습

## PART 1. 기존 프로젝트와 Docker 실행환경을 확인합니다

### 실습 1. 6일차 프로젝트를 다시 엽니다

새 프로젝트를 만들지 않고 6일차까지 작성한 프로젝트로 이동합니다. 기존 `.venv`는 **PC에서 코드를 수정·테스트할 때 계속 사용**하므로 삭제하지 않습니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
pwd
which python
git status --short
```

`pwd`가 `.../subject13_ai_platform`, `which python`이 해당 폴더의 `.venv/bin/python`으로 끝나는지 확인합니다.

7일차에는 기존 파일을 다음처럼 **유지**합니다.

```text
app/main.py                 (기존 1~6일차 API 및 Router 유지)
app/db.py, app/models.py    (DB 구조 변경 없음)
app/pipeline_api.py         (3일차 Export API 유지)
app/celery_app.py           (3일차 Celery 설정 유지)
app/worker_tasks.py         (3일차 Export 작업 그대로 재사용)
app/live_api.py             (4일차 AI Bridge 유지)
app/edge_api.py             (5일차 ONNX API 유지)
app/edge_device_api.py      (6일차 Pi 이벤트 API 유지)
```

### 실습 2. Docker와 Compose를 사용할 수 있는지 확인합니다

WSL Ubuntu 터미널에서 다음 명령을 순서대로 실행합니다.

```bash
docker --version
docker compose version
docker info --format '{{.ServerVersion}}'
```

Docker와 Compose 버전이 표시되고 Docker 서버 버전까지 출력되면 정상입니다. `docker --version`만 성공하고 `docker info`가 실패한다면 명령 프로그램은 설치되어 있지만 **Docker 엔진이 실행되고 있지 않거나 접근 권한이 없는 상태**일 수 있습니다.

Windows Docker Desktop을 사용하는 경우에는 Docker Desktop을 실행하고 WSL 통합이 활성화된 상태인지 확인합니다. 기관에서 제공하는 Docker Engine이 있다면 해당 설치 방법을 사용합니다. **Docker가 이미 정상 설치되어 있다면 새로 설치하지 않습니다.**

```bash
docker compose --help | head -15
```

여기서 `docker compose`는 Docker Compose v2 명령입니다. 예전 `docker-compose`와 혼용하지 않습니다.

### 실습 3. 컨테이너 빌드 전 필요한 소스와 자료를 자동 점검합니다

1~6일차까지 작성한 코드가 하나라도 빠져 있으면 컨테이너를 만든 뒤 어느 파일이 문제인지 찾기가 어려워집니다. 따라서 Docker 빌드 전에 **파일 준비 상태부터 확인**합니다.

**의사코드**

```text
교과 13 프로젝트의 최상위 경로 확인
             ↓
1~6일차 FastAPI · DB · Celery · Edge 파일 검사
             ↓
.env 존재 여부 확인
             ↓
교과9·10 JSON, 5일차 ONNX 파일 준비 상태 안내
             ↓
필요한 Export·백업 폴더 생성
             ↓
필수 소스가 준비되면 READY 출력
```

**파일: `scripts/17_check_day07_inputs.py`**

```python
"""7일차: Docker 빌드 전 1~6일차 프로젝트 준비 상태를 검사합니다."""
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]
REQUIRED = [
    "app/main.py", "app/db.py", "app/models.py", "app/repository.py",
    "app/records.py", "app/schemas.py", "app/celery_app.py",
    "app/worker_tasks.py", "app/data_pipeline.py", "app/pipeline_api.py",
    "app/live_api.py", "app/edge_api.py", "app/edge_device_api.py",
    "edge/runtime.py", "web/index.html", "web/db.html", "web/pipeline.html",
    "web/live.html", "web/edge.html", "web/pi.html",
    "scripts/02_create_tables.py", "scripts/03_import_records_to_db.py",
]
OPTIONAL_MODEL = ["artifacts/day05/tinycnn.onnx", "artifacts/day05/training_config.json"]
RECORDS = ["handoff/records/inspection_history.json", "handoff/records/action_jobs.json"]


def main() -> None:
    missing = [name for name in REQUIRED if not (ROOT / name).is_file()]
    for name in REQUIRED:
        print(("[MISSING]" if name in missing else "[OK]"), name)
    if not (ROOT / ".env").is_file():
        missing.append(".env")
        print("[MISSING] .env (2~6일차의 기존 설정을 유지하세요)")
    for name in OPTIONAL_MODEL:
        if not (ROOT / name).is_file():
            print("[NOTE] 5일차 ONNX 미리보기가 필요하면 준비:", name)
    for name in RECORDS:
        if not (ROOT / name).is_file():
            print("[NOTE] 신규 Docker DB에 기록을 재이관할 때 필요:", name)
    for directory in ["handoff/records", "artifacts/day05", "data/day03/exports", "backups/day07"]:
        (ROOT / directory).mkdir(parents=True, exist_ok=True)
    if missing:
        raise SystemExit("필수 파일이 없습니다. 누락된 1~6일차 실습을 확인하세요.")
    print("DAY07 PREFLIGHT: READY (필수 코드·.env 확인)")


if __name__ == "__main__":
    main()
```

코드에서 `REQUIRED`는 오늘 Docker 플랫폼을 실행하는 데 필요한 **앞선 실습 소스 목록**입니다. `OPTIONAL_MODEL`은 ONNX 이미지 추론을 사용할 때 필요한 파일이며, Docker 플랫폼 자체가 시작되는 조건은 아닙니다. `RECORDS`는 **새 Docker DB로 과거 기록을 재이관할 때** 필요합니다.

프로젝트 루트에서 실행합니다.

```bash
python -m scripts.17_check_day07_inputs
```

필수 파일이 모두 있으면 마지막에 다음 문장이 나옵니다.

```text
DAY07 PREFLIGHT: READY (필수 코드·.env 확인)
```

`[MISSING]`이 나오면 해당 일차에서 만든 파일이 있는지 확인하고, 필요한 코드를 먼저 완성합니다. 모델 파일이 없다는 안내를 받았다면 5일차 ONNX 미리보기만 보류하고, 별도의 제공받은 실제 모델 파일을 배치한 뒤 확인하면 됩니다.

**추가할 파일: `scripts/18_probe_compose.py`**

Docker 이미지는 PC의 코드를 그대로 자동 반영하지 않습니다. 따라서 17번에서 작성할 컨테이너 연결 점검 스크립트도 **이미지 빌드 전에** 아래처럼 미리 만들어 둡니다. 실제 실행은 실습 17에서 진행합니다.

**파일: `scripts/18_probe_compose.py`**

```python
"""7일차: 컨테이너 내부에서 API·PostgreSQL·Redis·Worker 연결 확인."""
from urllib.request import urlopen
import json

from app.container_api import probe_postgresql, probe_redis, probe_worker


def main() -> None:
    with urlopen("http://127.0.0.1:8000/health", timeout=5) as reply:
        health = json.load(reply)
    results = {
        "api_http": health.get("status") == "ok",
        "postgresql": probe_postgresql(),
        "redis": probe_redis(),
        "celery_worker_responding": probe_worker(),
    }
    for key, passed in results.items():
        print(f"{'PASS' if passed else 'FAIL'} : {key}")
    if not all(results.values()):
        raise SystemExit("연결에 실패한 서비스의 docker compose logs를 확인하세요.")
    print("DAY07 COMPOSE CONNECTIONS: PASS")


if __name__ == "__main__":
    main()
```

### 실습 4. 기존 PC 서버를 중지하고 PostgreSQL 기록을 백업합니다

**중요:** 이전 일차에서 `uvicorn` 또는 `celery worker`를 PC 터미널에서 실행 중이라면 **각 터미널에서 `Ctrl + C`**로 중지합니다. 그렇지 않으면 Docker FastAPI가 동일한 8000 포트를 사용할 수 없고, 오래된 Worker가 다른 Redis에서 별도로 동작할 수 있습니다.

PC에 설치된 PostgreSQL 서버는 **아직 중지하지 않습니다.** 기존 기록을 백업해야 하기 때문입니다.

```bash
mkdir -p backups/day07
pg_dump --version
pg_isready -h 127.0.0.1 -p 5432
```

2일차에 생성한 PostgreSQL이 PC의 `127.0.0.1:5432`에서 실행 중이라면 다음 명령으로 기록을 백업합니다.

```bash
pg_dump -h 127.0.0.1 -p 5432 \
  -U subject13_user -d subject13_platform \
  -Fc -f backups/day07/pre_docker.dump
```

실제 2일차 DB 이름·계정·포트가 다르면 자신의 환경에 맞는 값으로 변경합니다. 필요하면 기존 DB 계정의 비밀번호를 입력합니다. 비밀번호를 명령행 인자나 Git 파일에 기록하지 않습니다.

```bash
ls -lh backups/day07/pre_docker.dump
pg_restore -l backups/day07/pre_docker.dump | head -15
```

`pg_restore -l`에서 `TABLE`, `TABLE DATA` 등의 항목이 보이면 PostgreSQL 사용자 데이터베이스를 백업한 것입니다. 단순히 파일이 존재하는 것보다 **목록을 실제로 읽을 수 있는지** 확인하는 것이 중요합니다.

> **기존 PC PostgreSQL에 저장된 결과가 없거나 서버가 준비되지 않은 경우:** 오늘은 백업을 억지로 만들지 않습니다. 뒤의 실습 12에서 **1일차 JSON 재이관 방법(B)**을 선택합니다. 단, 그 방법으로는 2~6일차에 별도로 생성한 수동·실시간·Pi 기록을 모두 복원할 수 없습니다.

---

## PART 2. FastAPI·Worker가 사용할 공통 Docker 이미지를 작성합니다

### 실습 5. 7일차에 추가하는 파일 위치를 확인합니다

기존 파일은 삭제하지 않고 다음 파일을 **새로 추가**합니다.

```text
subject13_ai_platform/
├── Dockerfile                       [신규]
├── compose.yaml                     [신규]
├── .dockerignore                    [신규]
├── requirements.docker.txt         [신규]
├── app/
│   ├── main.py                      [기존 파일, Router 2줄만 추가]
│   └── container_api.py             [신규: 상태 API]
├── scripts/
│   ├── 17_check_day07_inputs.py     [신규: 사전 점검]
│   └── 18_probe_compose.py          [신규: 컨테이너 내부 연결 점검]
├── web/
│   └── containers.html              [신규: 서비스 상태 화면]
├── tests/
│   └── test_day07.py                [신규: 상태 API 자동 테스트]
├── docs/
│   └── day07_compose.md             [신규: Compose 구조·복구 메모]
├── handoff/records/                [기존 결과, 이미지에 포함하지 않음]
├── artifacts/day05/               [기존 모델, 이미지에 포함하지 않음]
├── data/day03/exports/            [기존 결과, 공유 폴더로 연결]
└── backups/day07/                 [DB 백업, Git에 포함하지 않음]
```

오늘도 **1~6일차 `app/main.py`, `app/models.py`, `app/db.py`, `requirements.txt`를 새 내용으로 통째로 교체하지 않습니다.** 기존 Python 가상환경의 패키지 목록은 학습·검증을 위해 유지하고, Docker 전용으로 **별도의** `requirements.docker.txt`를 작성합니다.

### 실습 6. Docker 전용 Python 패키지 목록을 작성합니다

**이번 파일은 무엇을 하나요?** FastAPI와 Celery Worker가 공동으로 필요한 패키지만 설치하도록 합니다. 교과 10 YOLO, 5일차 PyTorch 모델 학습과 Raspberry Pi의 GPIO 라이브러리는 이번 PC Docker 이미지에서 실행하지 않으므로 포함하지 않습니다.

**파일을 만들기 전에 처리 순서를 확인합니다.**

```text
기존 1~6일차 API가 import하는 Python 패키지 확인
           ↓
FastAPI / SQLAlchemy / Psycopg 준비
           ↓
Redis / Celery / PyArrow 준비
           ↓
ONNX Runtime CPU / Pillow / NumPy 준비
           ↓
학습용 PyTorch·YOLO·GPIO는 제외
           ↓
Docker 이미지에 이 목록 설치
```

**파일: `requirements.docker.txt`**

```text
# 교과 13일차 1~6 FastAPI/DB/Worker/ONNX CPU 실행용 패키지.
# 학습용 PyTorch·YOLO·OpenCV는 교과 9/10/5일차의 기존 환경에 둡니다.
fastapi>=0.115,<1
uvicorn[standard]>=0.30,<1
pydantic>=2,<3
httpx>=0.27,<1
pytest>=8,<10
SQLAlchemy>=2.0,<3
psycopg[binary]>=3.2,<4
python-dotenv>=1.0,<2
celery>=5.4,<6
redis>=5,<7
pyarrow>=18,<25
numpy>=1.26,<3
Pillow>=10,<13
onnxruntime>=1.18,<2
python-multipart>=0.0.9,<1
```

`psycopg[binary]`는 PostgreSQL 연결 드라이버이며, `onnxruntime`은 5일차 PC 미리보기 API가 시작될 때 사용하는 CPU 추론 패키지입니다. `pyarrow`는 3일차 Silver Parquet을 만들기 위해 필요합니다. 기존 `.venv`에 있는 모든 패키지를 기계적으로 복사하지 않습니다.

### 실습 7. Dockerfile을 작성합니다

Dockerfile은 **기존 Python 앱을 어떤 운영체제·Python·패키지 조합으로 실행할지** 기록하는 파일입니다.

**의사코드**

```text
Python 3.12 이미지 사용
       ↓
컨테이너 작업 폴더 /app 지정
       ↓
ONNX Runtime CPU에 필요한 시스템 라이브러리 설치
       ↓
Docker 전용 requirements 설치
       ↓
기존 app/ · edge/ · scripts/ · web/ · tests/ 등 복사
       ↓
FastAPI 기본 실행 명령 등록
```

**파일: `Dockerfile`**

```dockerfile
# 교과 13 · 7일차: PC FastAPI + Celery Worker 공용 이미지 (CPU/AMD64 중심)
FROM python:3.12-slim-bookworm
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app
WORKDIR /app
# ONNX Runtime CPU가 사용하는 OpenMP 런타임
RUN apt-get update && apt-get install -y --no-install-recommends libgomp1 \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.docker.txt ./requirements.docker.txt
RUN python -m pip install --no-cache-dir -r requirements.docker.txt
COPY app/ ./app/
COPY edge/ ./edge/
COPY scripts/ ./scripts/
COPY web/ ./web/
COPY device/ ./device/
COPY tests/ ./tests/
COPY docs/ ./docs/
COPY README.md ./README.md
EXPOSE 8000
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**코드를 읽어 봅니다.** `FROM`은 Python 실행 기반 이미지, `WORKDIR /app`은 컨테이너 안의 프로젝트 위치입니다. `COPY`로 소스파일을 이미지에 포함하지만, **학습 모델·개인 영상·과거 결과 JSON은 포함하지 않습니다.** `CMD`는 FastAPI의 기본 실행 명령입니다. 나중에 Worker 컨테이너는 같은 이미지에서 다른 `command`를 실행합니다.

`--host 0.0.0.0`은 **컨테이너 내부**에서 Uvicorn이 요청을 받는 주소입니다. PC 외부에 실제로 공개되는 범위는 뒤의 Compose `ports` 설정에서 별도로 제한합니다.

### 실습 8. `.dockerignore`로 이미지에 들어가지 않을 파일을 제외합니다

1~6일차의 `.gitignore`와 `.dockerignore`는 역할이 다릅니다. `.gitignore`는 Git 추적을 제한하고, `.dockerignore`는 Docker **빌드할 때** 컴퓨터에서 이미지 빌드 환경으로 전달할 파일을 제한합니다.

**의사코드**

```text
소스코드(app, web, edge, scripts)는 이미지에 포함
       ↓
.env / .venv / 모델 / 원본 영상 / 백업 / 결과 JSON 제외
       ↓
Docker 이미지 빌드 시 필요한 파일만 사용
```

**파일: `.dockerignore`**

```gitignore
.git
.venv
.env
.env.*
!.env.example
__pycache__
.pytest_cache
*.py[cod]
data/
artifacts/
handoff/
deploy/
reports/
backups/
*.pt
*.pth
*.joblib
*.onnx
*.engine
*.mp4
*.avi
*.zip
```

`.env`가 이미지에 들어가면 비밀번호나 장치 인증 키가 이미지 안에 남을 수 있습니다. `.dockerignore`에 반드시 등록합니다. 대신 실제 실행할 때는 **Compose가 `.env` 값을 읽어 환경변수로 주입**합니다. 비밀값은 지금 작성하는 Dockerfile에도 직접 입력하지 않습니다.

---

## PART 3. Docker Compose로 네 서비스를 하나로 구성합니다

### 실습 9. Compose가 사용할 설정을 확인합니다

2일차부터 사용한 `.env`에는 실제 DB 비밀번호가, 6일차에는 Raspberry Pi 전송 키가 추가되어 있습니다. **기존 `.env`를 새 파일로 덮어쓰지 않습니다.**

프로젝트 루트에서 `.env`가 있는지 확인합니다.

```bash
ls -lh .env .env.example
```

설정 파일에는 다음 종류의 값이 유지되어야 합니다. 아래의 비밀번호·키는 **예시 자리표시자이므로 그대로 붙여 넣으면 안 됩니다.** 이전 일차에 만든 본인의 실제 값을 유지합니다.

```dotenv
# 2일차에서 사용한 기존 값 — 실제 값 유지
DB_HOST=127.0.0.1
DB_PORT=5432
DB_NAME=subject13_platform
DB_USER=subject13_user
DB_PASSWORD=본인의_기존_실제_DB_비밀번호

# 3일차에서 사용한 기존 값 — PC 직접 실행용
REDIS_URL=redis://127.0.0.1:6379/0

# 6일차에서 생성한 실제 비밀키를 그대로 사용
DAY06_DEVICE_API_KEY=기존_6일차_실제_장치_키

# 7일차 추가: 기본은 이 PC만 웹 접속 허용
PLATFORM_BIND_HOST=127.0.0.1
```

기존 값은 **수정하지 않고** `.env` 맨 아래에 다음 한 줄만 추가해도 됩니다.

```dotenv
PLATFORM_BIND_HOST=127.0.0.1
```

`.env.example`에도 위 한 줄만 추가합니다. `.env.example`에는 실제 비밀번호와 키를 넣지 않습니다.

`DAY06_DEVICE_API_KEY`가 없다면 **6일차에서 생성한 키를 다시 확인**합니다. 새 키를 임의로 바꾸면 Raspberry Pi의 기존 `.env`와 달라져 정상 장치 요청도 HTTP 401이 될 수 있습니다.

### 실습 10. `compose.yaml`을 작성합니다

오늘 사용할 네 개의 서비스는 하나의 `compose.yaml`로 정의합니다.

**의사코드**

```text
PostgreSQL 서비스 db 정의
        ├─ DB 이름·계정·비밀번호 환경변수 설정
        ├─ 데이터를 pg_data 볼륨에 보관
        └─ pg_isready로 Healthcheck
             ↓
Redis 서비스 redis 정의
        ├─ AOF 활성화·redis_data 볼륨
        └─ redis-cli ping Healthcheck
             ↓
FastAPI 서비스 api 정의
        ├─ 공통 Docker 이미지 빌드
        ├─ DB_HOST=db / REDIS_URL=redis://redis:6379/0
        ├─ 기존 결과 JSON·ONNX·Export 폴더 연결
        └─ 호스트 127.0.0.1:8000에 웹 공개
             ↓
Celery Worker 서비스 worker 정의
        ├─ api와 같은 Docker 이미지 사용
        ├─ 3일차 Celery 실행 명령 사용
        └─ FastAPI와 같은 DB·Redis·Export 폴더 사용
```

**파일: `compose.yaml`**

```yaml
# 교과 13 · 7일차: PC 중앙 플랫폼. 모델 학습 및 Raspberry Pi·Jetson은 이 Compose 밖에서 실행합니다.
services:
  db:
    image: postgres:16-bookworm
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME:-subject13_platform}
      POSTGRES_USER: ${DB_USER:-subject13_user}
      POSTGRES_PASSWORD: ${DB_PASSWORD:? .env 파일에 실제 DB_PASSWORD를 설정하세요}
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U \"$${POSTGRES_USER}\" -d \"$${POSTGRES_DB}\""]
      interval: 5s
      timeout: 3s
      retries: 12
    networks: [platform]

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 12
    networks: [platform]

  api:
    image: subject13-platform:day07
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    depends_on:
      db: {condition: service_healthy}
      redis: {condition: service_healthy}
    environment:
      DB_HOST: db
      DB_PORT: "5432"
      DB_NAME: ${DB_NAME:-subject13_platform}
      DB_USER: ${DB_USER:-subject13_user}
      DB_PASSWORD: ${DB_PASSWORD:? .env 파일에 실제 DB_PASSWORD를 설정하세요}
      REDIS_URL: redis://redis:6379/0
      DAY06_DEVICE_API_KEY: ${DAY06_DEVICE_API_KEY:? .env 파일에 6일차 장치 키를 설정하세요}
      DAY04_OCR_URL: http://host.docker.internal:8009
      DAY04_ACTION_URL: http://host.docker.internal:8010
      PYTHONPATH: /app
    ports:
      - "${PLATFORM_BIND_HOST:-127.0.0.1}:8000:8000"
    volumes:
      - ./handoff/records:/app/handoff/records:ro
      - ./artifacts/day05:/app/artifacts/day05:ro
      - ./data/day03/exports:/app/data/day03/exports
    extra_hosts:
      - "host.docker.internal:host-gateway"
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=3)"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 10s
    networks: [platform]

  worker:
    image: subject13-platform:day07
    restart: unless-stopped
    command: ["celery", "-A", "app.celery_app:celery_app", "worker", "--loglevel=info", "--pool=solo"]
    depends_on:
      db: {condition: service_healthy}
      redis: {condition: service_healthy}
    environment:
      DB_HOST: db
      DB_PORT: "5432"
      DB_NAME: ${DB_NAME:-subject13_platform}
      DB_USER: ${DB_USER:-subject13_user}
      DB_PASSWORD: ${DB_PASSWORD:? .env 파일에 실제 DB_PASSWORD를 설정하세요}
      REDIS_URL: redis://redis:6379/0
      PYTHONPATH: /app
    volumes:
      - ./handoff/records:/app/handoff/records:ro
      - ./artifacts/day05:/app/artifacts/day05:ro
      - ./data/day03/exports:/app/data/day03/exports
    networks: [platform]

networks:
  platform:
    driver: bridge
volumes:
  pg_data:
  redis_data:
```

**중요한 설정을 확인합니다.**

- `db`, `redis`는 Docker 네트워크 안에서만 실행합니다. `ports`를 지정하지 않았으므로 개발 PC 외부에 5432·6379 포트를 공개하지 않습니다.
- `pg_data`, `redis_data`는 컨테이너를 다시 만들더라도 보존되는 **Docker named volume**입니다.
- FastAPI와 Worker의 `DB_HOST=db`, `REDIS_URL=redis://redis:6379/0`은 PC `.env`의 `127.0.0.1` 설정과 구분됩니다.
- 두 Python 컨테이너의 `./data/day03/exports` 바인드 마운트는 **같은 디렉터리**이므로 Worker가 만든 파일을 PC에서도 볼 수 있습니다.
- `./handoff/records`와 `./artifacts/day05`는 `:ro`, 즉 **읽기 전용**으로 마운트합니다. 원본을 컨테이너에서 임의로 고치지 않습니다.
- `depends_on: condition: service_healthy`는 DB와 Redis가 Healthcheck를 통과할 때까지 API·Worker 시작 순서를 제어합니다. **완전한 장애 복구나 개별 작업의 성공을 보장하는 기능은 아닙니다.**
- `host.docker.internal`은 **컨테이너에서 PC에서 실행 중인 4일차 AI Bridge에 접근할 때 사용하는 주소 후보**입니다. 실제로 접근되려면 호스트 서비스의 Listen 주소와 Docker/WSL 네트워크 설정이 맞아야 합니다. 4일차 AI 서버가 꺼져 있어도 이 기본 Compose 플랫폼은 시작할 수 있습니다.

### 실습 11. Compose 설정을 먼저 검증합니다

**먼저 YAML을 검사합니다.**

```bash
docker compose config --quiet
```

설정이 정상이면 보통 아무 메시지도 출력되지 않고 종료합니다. 오류가 나오면 `.env`의 `DB_PASSWORD`와 `DAY06_DEVICE_API_KEY`가 설정되어 있는지 확인합니다. **`docker compose config`를 옵션 없이 실행하면 비밀값이 화면에 표시될 수 있으므로 사용하지 않습니다.**

서비스 이름을 확인합니다.

```bash
docker compose config --services
```

정상이라면 다음 서비스가 보입니다. 순서는 환경에 따라 다를 수 있습니다.

```text
db
redis
api
worker
```

Docker 이미지에는 **빌드 시점에 존재하는 소스코드만** 포함됩니다. 그러므로 앞에서 정의한 Compose 형식만 먼저 검사하고, 이번 7일차의 새 `app/container_api.py`와 웹 화면까지 작성한 뒤 **실습 16에서 이미지를 빌드**합니다.

단, **방법 B처럼 DB 초기화 스크립트를 먼저 컨테이너에서 실행해야 하는 학생**은 실습 12에서 한 번 미리 빌드합니다. 이후 7일차 코드가 모두 추가되면 실습 16에서 다시 빌드하며, 정상적으로 캐시가 작동한다면 설치 패키지를 매번 다시 내려받지 않습니다.

---

## PART 4. PostgreSQL·Redis를 실행하고 이전 기록을 복원합니다

### 실습 12. PostgreSQL과 Redis 컨테이너부터 실행합니다

여기서는 **DB와 Redis만 먼저** 실행합니다. 기존 PC PostgreSQL과 Docker PostgreSQL을 혼동하지 않기 위해 저장소 이전이 끝난 뒤 FastAPI를 실행합니다.

```bash
docker compose up -d db redis
docker compose ps
```

두 서비스가 `Up` 또는 `healthy`로 나타나는지 확인합니다.

```bash
docker compose exec db sh -c 'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
docker compose exec redis redis-cli ping
```

PostgreSQL에는 `accepting connections`, Redis에는 `PONG`이 나오면 각각 서버 응답을 확인한 것입니다. PostgreSQL의 DB 이름과 계정은 Compose의 `.env`를 통해 설정되었습니다.

#### 방법 A — 2~6일차 PostgreSQL 기록 전체를 이어서 사용하는 경우

앞의 실습 4에서 `pre_docker.dump` 백업을 만들었다면 **새로 만든 빈 Docker DB에 한 번만 복원**합니다. DB가 이미 테이블과 데이터를 가지고 있는 경우에는 이 복원 명령을 반복하지 않습니다.

**의사코드**

```text
PC PostgreSQL의 기존 Dump 파일 확인
           ↓
Docker PostgreSQL 컨테이너가 비어 있는지 확인
           ↓
Dump를 Docker PostgreSQL로 복원
           ↓
기존 inference_events·source·model 테이블 확인
           ↓
2~6일차 기록이 유지되었는지 SQL로 검사
```

프로젝트 루트에서 실행합니다.

```bash
ls -lh backups/day07/pre_docker.dump

docker compose exec -T db sh -c \
  'pg_restore -U "$POSTGRES_USER" -d "$POSTGRES_DB" --no-owner --no-privileges' \
  < backups/day07/pre_docker.dump
```

`-T`는 표준입력으로 들어오는 백업 파일을 컨테이너에 전달할 때 가상 터미널을 비활성화합니다. 복원이 성공하면 기존 DB 테이블과 그 안의 데이터가 새 Docker PostgreSQL로 옮겨집니다.

```bash
docker compose exec -T db sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT source_mode, COUNT(*) FROM inference_events GROUP BY source_mode ORDER BY source_mode;"'
```

`recorded_result`, `manual_entry`, `live_inference` 등의 실제 기록 수를 확인합니다. 학생마다 기록 개수가 다를 수 있으므로 특정 숫자를 외울 필요는 없습니다. **기존 PC DB에 있던 결과가 계속 남아 있는지**가 중요합니다.

> `pg_restore`에서 테이블이 이미 존재한다는 오류가 발생하면 **이전에 Docker DB가 만들어졌는지 먼저 확인**합니다. 기존 데이터를 지우기 위해 `docker compose down -v`를 실행하지 않습니다. 백업 상태와 충돌을 확인해야 합니다.

#### 방법 B — 기존 DB 백업이 없고 1일차 원본 JSON만 있는 경우

이 방법은 **실습 12의 방법 A를 실행하지 않은 경우에만** 사용합니다. 1일차의 검사·동작인식 JSON에서 새 Docker DB를 초기화합니다.

```bash
ls -lh handoff/records/inspection_history.json handoff/records/action_jobs.json

# DB 초기화 스크립트를 실행하기 위해 기본 이미지를 먼저 빌드합니다.
docker compose build api

docker compose run --rm api python -m scripts.02_create_tables
docker compose run --rm api python -m scripts.03_import_records_to_db
```

기존 2일차에 작성한 테이블 생성·원본 결과 이관 코드를 **컨테이너 안에서 그대로 실행**합니다. 아래 명령을 다시 실행하면 이미 저장된 공통 이벤트는 `event_id`를 기준으로 중복을 건너뛰는지 확인할 수 있습니다.

```bash
docker compose run --rm api python -m scripts.03_import_records_to_db
```

방법 B로는 **기존 2~6일차의 수동 CRUD·실시간 OCR·Pose·Raspberry Pi 기록이 자동으로 복원되지 않습니다.** 그 데이터까지 필요하면 이전 DB Dump를 확보해 방법 A로 이전하는 것이 적절합니다.

#### 두 방법 모두 — DB 테이블과 기록을 확인합니다

```bash
docker compose exec -T db sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT COUNT(*) AS event_count FROM inference_events;"'

docker compose exec -T db sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT task, decision, COUNT(*) FROM inference_events GROUP BY task, decision ORDER BY task, decision;"'
```

기존 교과 9·10 또는 4~6일차 결과가 들어 있는지 확인합니다. 지금 확인하는 DB는 **새 Docker PostgreSQL**이며, PC에서 설치했던 옛 PostgreSQL의 테이블을 직접 조회하는 명령이 아닙니다.

---

## PART 5. 기존 FastAPI에 Docker 서비스 상태 확인을 추가합니다

### 실습 13. PostgreSQL·Redis·Worker 연결 검사 코드를 작성합니다

FastAPI만 열렸다고 DB와 Redis, Worker까지 정상이라는 뜻은 아닙니다. 오늘은 각각을 직접 확인하는 API를 추가합니다.

**의사코드**

```text
브라우저가 /api/v1/containers/status 요청
        ↓
PostgreSQL에 SELECT 1 → 응답 여부 기록
        ↓
Redis에 PING → 응답 여부 기록
        ↓
Celery Worker에 inspect ping → 응답 여부 기록
        ↓
각 결과를 Boolean으로 정리
        ↓
모두 연결되면 ok, 하나라도 미확인이면 degraded
        ↓
비밀번호·장치 키·내부 연결 문자열은 반환하지 않음
```

**파일: `app/container_api.py`**

```python
"""7일차: Docker Compose 서비스 준비 상태와 운영 화면."""
from pathlib import Path

from fastapi import APIRouter
from fastapi.responses import FileResponse

ROOT = Path(__file__).resolve().parents[1]
router = APIRouter(tags=["day07-compose"])


def probe_postgresql() -> bool:
    """새 Session에서 PostgreSQL SELECT 1을 실행합니다."""
    try:
        from sqlalchemy import text
        from app.db import SessionLocal
        with SessionLocal() as db:
            return db.execute(text("SELECT 1")).scalar_one() == 1
    except Exception:
        return False


def probe_redis() -> bool:
    """Redis의 PING 명령으로 현재 연결만 확인합니다."""
    try:
        import os
        import redis
        client = redis.Redis.from_url(
            os.getenv("REDIS_URL", "redis://127.0.0.1:6379/0"),
            socket_connect_timeout=1, socket_timeout=1,
        )
        try:
            return bool(client.ping())
        finally:
            client.close()
    except Exception:
        return False


def probe_worker() -> bool:
    """Celery inspect ping 응답을 확인합니다(작업 성공을 뜻하지 않음)."""
    try:
        from app.celery_app import celery_app
        replies = celery_app.control.ping(timeout=2)
        return bool(replies)
    except Exception:
        return False


@router.get("/containers", include_in_schema=False)
def container_page():
    return FileResponse(ROOT / "web" / "containers.html")


@router.get("/api/v1/containers/status")
def container_status():
    checks = {
        "postgresql": probe_postgresql(),
        "redis": probe_redis(),
        "celery_worker_responding": probe_worker(),
    }
    return {
        "status": "ok" if all(checks.values()) else "degraded",
        "services": checks,
        "note": "Worker 응답은 현재 생존 확인이며, 내보내기 성공은 별도 작업 결과로 검증합니다.",
    }
```

`probe_postgresql()`은 기존 2일차 `SessionLocal`을 사용하고, `probe_redis()`는 3일차 Redis에 실제 `PING`을 보냅니다. `probe_worker()`는 Celery Worker에 **상태 질의에 응답하는지** 확인합니다. Worker가 응답하더라도 실제 Export 작업이 성공했다는 뜻은 아닙니다. 내보내기 성공 여부는 뒤에서 작업 ID로 확인합니다.

서버와 Redis 정보가 일시적으로 끊기면 `status`가 `degraded`가 됩니다. 비밀번호나 내부 연결 문자열은 API 결과에 포함시키지 않습니다.

### 실습 14. 기존 `app/main.py`에 7일차 Router를 추가합니다

**수정 파일: `app/main.py`**

1~6일차의 기존 라우터와 `app = FastAPI(...)`를 **그대로 유지**합니다. 파일 **맨 아래**에 다음 두 줄만 **한 번** 추가합니다.

```python
from app.container_api import router as container_router
app.include_router(container_router)
```

기존의 `db_router`, `pipeline_router`, `live_router`, `edge_router`, `edge_device_router`를 삭제하지 않습니다. 이 단계에서 DB 모델이나 `InferenceEvent`의 필드도 수정하지 않습니다.

### 실습 15. Docker 통합 상태를 보여주는 웹 화면을 작성합니다

이번에는 FastAPI가 응답한 세 서비스의 준비 상태를 브라우저에서 확인합니다. 기존 `web/index.html`, `web/db.html`, `web/pipeline.html`, `web/live.html`, `web/edge.html`, `web/pi.html`은 그대로 둡니다.

**의사코드**

```text
브라우저 /containers 접속
       ↓
GET /api/v1/containers/status 실행
       ↓
PostgreSQL·Redis·Worker 상태 JSON 수신
       ↓
각 항목을 연결 확인/확인 필요로 표시
       ↓
이전 일차의 /db, /pipeline, /live, /edge-device 이동 링크 제공
```

**파일: `web/containers.html`**

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>교과 13 · 7일차 Compose 상태</title>
  <style>
    body {font-family:system-ui,sans-serif;max-width:900px;margin:32px auto;padding:0 16px;color:#24364a}
    .panel {border:1px solid #cbd5e1;border-radius:10px;padding:18px;margin-top:16px}
    .ok {color:#166534;font-weight:bold}.fail{color:#b91c1c;font-weight:bold}
    a{color:#075e9e;margin-right:10px}button{padding:9px 14px;cursor:pointer}
    li{margin:9px 0}pre{background:#f1f5f9;white-space:pre-wrap;padding:12px}
  </style>
</head>
<body>
  <h1>교과 13 · 7일차 Docker Compose</h1>
  <p>1~6일차의 기존 FastAPI에 PostgreSQL·Redis·Celery Worker를 컨테이너로 연결합니다.</p>
  <section class="panel">
    <h2>서비스 연결 상태</h2>
    <button id="refresh" type="button">상태 다시 확인</button>
    <ul id="checks"><li>확인 중입니다.</li></ul>
    <p id="message" role="status"></p>
    <pre id="raw">{}</pre>
  </section>
  <section class="panel">
    <h2>이전 실습 화면</h2>
    <p><a href="/">1일차 공통 기록</a><a href="/db">2일차 DB</a>
    <a href="/pipeline">3일차 Celery</a><a href="/live">4일차 실제 추론</a>
    <a href="/api/v1/edge/page">5일차 ONNX</a><a href="/edge-device">6일차 Pi</a><a href="/docs">Swagger</a></p>
    <p>4일차의 기존 OCR/Pose 서비스는 별도 실행환경이므로, Docker 서버만 켠 상태에서는 연결되지 않을 수 있습니다.</p>
  </section>
  <script>
    const labels={postgresql:"PostgreSQL",redis:"Redis",celery_worker_responding:"Celery Worker 응답"};
    async function refresh(){
      const checks=document.getElementById("checks"), message=document.getElementById("message");
      checks.replaceChildren(); message.textContent="상태를 확인하고 있습니다.";
      try{
        const response=await fetch("/api/v1/containers/status",{cache:"no-store"});
        if(!response.ok) throw new Error(`HTTP ${response.status}`);
        const data=await response.json();
        for(const [name,ready] of Object.entries(data.services)){
          const li=document.createElement("li");
          li.textContent=`${labels[name] || name}: ${ready ? "연결 확인" : "연결 확인 필요"}`;
          li.className=ready?"ok":"fail"; checks.append(li);
        }
        message.textContent=data.note;
        document.getElementById("raw").textContent=JSON.stringify(data,null,2);
      }catch(error){
        message.textContent=`연결 오류: ${error.message}`;
        document.getElementById("raw").textContent="{}";
      }
    }
    document.getElementById("refresh").addEventListener("click",refresh);
    refresh();
  </script>
</body>
</html>
```

`fetch()`는 새 상태 API를 호출합니다. `textContent`로 결과를 표시하므로 응답 문자열을 HTML 코드로 실행하지 않습니다. 실제 서비스 상태는 변할 수 있으므로 **상태 다시 확인** 버튼으로 재조회할 수 있습니다.

### 실습 16. FastAPI와 Celery Worker를 컨테이너로 실행합니다

앞에서 DB·Redis 시작과 데이터 복원이 끝났습니다. 이제 **방금 작성한 7일차 API와 웹 코드까지 포함되도록** Python 이미지를 빌드한 뒤 API와 Worker를 실행합니다. 실습 12에서 한 번 빌드했다면 이 명령은 변경된 소스만 새 이미지에 반영합니다.

```bash
docker compose build api
docker compose up -d --force-recreate api worker
docker compose ps
```

```bash
docker image ls subject13-platform
```

`subject13-platform:day07` 이미지가 확인되면 오늘 작성한 코드가 포함된 이미지가 만들어진 것입니다. Docker Hub와 패키지 저장소 접속이 제한된 학습실에서는 **승인된 사전 이미지·내부 패키지 미러·호환 Wheel**이 필요합니다. 교과 9·10 전체 GPU 모델을 다운로드하는 과정은 아닙니다.

잠시 후 네 서비스가 모두 `Up` 상태인지 확인합니다. `api`에는 `healthy` 표시가 나타날 수 있습니다. `worker`는 개별 Docker Healthcheck를 정의하지 않았으므로, 실제 Worker 응답은 다음 실습에서 확인합니다.

로그에서 실행 시작 메시지를 확인합니다.

```bash
docker compose logs --tail=40 api
docker compose logs --tail=40 worker
```

Worker 로그에 `day03.export_events`와 `ready`가 표시되면 Celery 작업 수신 준비 상태를 확인한 것입니다.

브라우저에서 다음 주소를 차례대로 엽니다.

```text
http://127.0.0.1:8000/
http://127.0.0.1:8000/db
http://127.0.0.1:8000/pipeline
http://127.0.0.1:8000/edge-device
http://127.0.0.1:8000/containers
http://127.0.0.1:8000/docs
```

- `/`는 1일차 **기존 결과 JSON**을 보여줍니다.
- `/db`는 2일차 PostgreSQL 기록을 보여줍니다.
- `/pipeline`은 3일차 Redis·Celery Export 화면입니다.
- `/edge-device`는 6일차 Raspberry Pi에서 저장한 결과를 조회합니다.
- `/containers`는 오늘 추가한 중앙 서비스 연결 상태 화면입니다.
- `/docs`에서는 기존 API와 7일차 새 API가 함께 등록되어 있는지 확인합니다.

기존 ONNX 모델 파일이 정상 준비되었다면 `http://127.0.0.1:8000/api/v1/edge/page`에서 5일차 이미지 미리보기도 확인할 수 있습니다. ONNX 모델 파일이 없을 때 미리보기 API가 503을 반환하는 것은 정상적인 준비물 안내입니다.

---

## PART 6. 실제 연결·비동기 작업·데이터 보존을 검증합니다

### 실습 17. 컨테이너 내부의 네 서비스 연결을 검사합니다

실습 3에서 **이미 이미지에 포함되도록 미리 작성한 `scripts/18_probe_compose.py`**를 사용합니다. 이 코드는 이미 실행 중인 FastAPI 컨테이너 **안에서** HTTP·SQL·Redis·Worker 응답을 한 번에 확인합니다.

**의사코드**

```text
FastAPI의 /health에 HTTP 요청
          ↓
기존 SQLAlchemy로 PostgreSQL SELECT 1
          ↓
Redis PING
          ↓
Celery Worker inspect ping
          ↓
네 결과를 각각 PASS/FAIL로 출력
          ↓
하나라도 FAIL이면 종료 코드로 오류 표시
```

이제 **PC 터미널**에서 다음 명령을 실행합니다.

```bash
docker compose exec api python -m scripts.18_probe_compose
```

정상이라면 다음과 같이 나옵니다.

```text
PASS : api_http
PASS : postgresql
PASS : redis
PASS : celery_worker_responding
DAY07 COMPOSE CONNECTIONS: PASS
```

Worker가 아직 시작 중이면 `celery_worker_responding`이 `FAIL`일 수 있습니다. Worker 로그에서 `ready`를 확인하고 잠시 후 다시 실행합니다. Redis 연결은 성공했지만 Worker만 실패한다면 **`docker compose logs worker`를 우선 확인**합니다.

브라우저에서도 `http://127.0.0.1:8000/containers`를 열어 같은 상태를 확인합니다.

### 실습 18. 3일차 Celery Export를 실제 Docker Worker로 실행합니다

3일차에 이미 작성한 `/api/v1/pipeline/exports` API를 **새로 작성하지 않고** 그대로 사용합니다. 오늘은 그 작업을 컨테이너의 Worker가 처리합니다.

```text
브라우저 /pipeline
       ↓ POST /api/v1/pipeline/exports
컨테이너 api
       ↓
Redis Broker
       ↓
컨테이너 worker
       ↓
Docker PostgreSQL의 recorded_result 조회
       ↓
PC data/day03/exports/에 JSONL·Parquet·Gold 생성
       ↓
Redis에 SUCCESS 기록
       ↓
브라우저에서 작업 ID로 상태 조회
```

먼저 브라우저에서 다음 주소를 엽니다.

```text
http://127.0.0.1:8000/pipeline
```

3일차와 마찬가지로 `task=전체`, `limit=100` 정도로 Export를 등록하고 작업 ID를 확인합니다. 터미널에서 직접 실행하려면 다음 명령을 사용할 수 있습니다.

```bash
curl -sS -X POST 'http://127.0.0.1:8000/api/v1/pipeline/exports' \
  -H 'Content-Type: application/json' \
  -d '{"task": null, "limit": 100}' | python -m json.tool
```

반환되는 `job_id`는 **실행할 때마다 달라집니다.** 화면에 나온 작업 ID를 복사한 뒤 아래의 `실제_작업ID`를 변경하여 조회합니다.

```bash
curl -sS 'http://127.0.0.1:8000/api/v1/pipeline/jobs/실제_작업ID' \
  | python -m json.tool
```

`PENDING`은 **아직 완료되지 않았거나 결과 정보가 없는 상태**일 수 있습니다. `STARTED`라면 Worker가 작업을 처리 중이며, 최종적으로 `SUCCESS`와 `result` 안의 `manifest.json` 경로를 확인해야 완료입니다.

3일차와 같은 검증 스크립트를 컨테이너 안에서 다시 실행합니다.

```bash
docker compose exec api python -m scripts.06_inspect_export
```

마지막에 다음 메시지가 표시되어야 합니다.

```text
검증 결과: PASS (Manifest = Parquet = Gold 행 수)
```

PC의 기존 프로젝트 폴더에서도 파일을 확인합니다.

```bash
find data/day03/exports -maxdepth 2 -type f \( -name 'bronze_events.jsonl' -o -name 'silver_events.parquet' -o -name 'gold_summary.json' -o -name 'manifest.json' \) | sort
```

**중요:** 3일차 Export 함수는 기본적으로 `source_mode='recorded_result'` 기록을 대상으로 합니다. **4일차 실제 추론이나 6일차 Pi 실시간 결과를 모두 Export한다고 오해하지 않습니다.** 실시간 결과를 별도로 내보내는 기능은 이후 필요한 범위를 정해 확장해야 합니다.

### 실습 19. 기존 PostgreSQL 기록이 재시작 후 유지되는지 확인합니다

앞에서 데이터베이스 기록 수를 확인했습니다. 컨테이너를 다시 시작해도 기록이 유지되는지 같은 SQL로 비교합니다.

**재시작 전 확인**

```bash
docker compose exec -T db sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT COUNT(*) AS before_restart FROM inference_events;"'
```

**네 서비스를 재시작**

```bash
docker compose restart
docker compose ps
```

서비스가 다시 시작될 때까지 기다린 후 다시 확인합니다.

```bash
docker compose exec -T db sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT COUNT(*) AS after_restart FROM inference_events;"'
```

재시작 전후의 건수가 같아야 합니다. 3일차 Export 결과 폴더도 PC에 그대로 남아 있는지 확인합니다.

```bash
find data/day03/exports -maxdepth 2 -name 'manifest.json' | sort
```

이 결과는 **PostgreSQL named volume과 파일 바인드 마운트가 데이터를 보존한다는 것**을 확인하는 실습입니다. 반대로 Redis에 저장된 과거 작업 상태는 3일차의 `result_expires=3600` 등의 설정 때문에 만료될 수 있습니다. **오래된 작업 상태와 Export 파일 보존을 같은 개념으로 보지 않습니다.**

### 실습 20. 컨테이너를 중지하고 다시 실행하는 방법을 확인합니다

기존 기록을 남긴 상태로 **컨테이너만** 내리고 다시 실행하는 연습입니다.

```bash
docker compose down
docker compose up -d
docker compose ps
```

`docker compose down`은 컨테이너와 Compose 네트워크를 정리하지만, 기본적으로 이름 있는 볼륨은 삭제하지 않습니다. Docker 이미지와 데이터 파일도 남습니다.

**다음 명령은 실행하지 않습니다.**

```text
docker compose down -v   ← 오늘 실행 금지: PostgreSQL·Redis 볼륨까지 삭제할 수 있습니다.
docker volume prune      ← 오늘 실행 금지: 다른 프로젝트 볼륨도 삭제 위험이 있습니다.
```

다시 `/db`, `/pipeline`, `/containers`를 열어 기록과 서비스가 정상인지 확인합니다. 이후에도 수업을 계속할 예정이므로 최종적으로 네 서비스가 실행된 상태로 두거나, 학습실 운영 방침에 따라 `docker compose stop`으로 종료할 수 있습니다.

### 실습 21. Docker DB를 다시 백업합니다

기존 DB를 Docker로 이전하고 새로운 Container DB에 기록이 남아 있는지 확인했다면, **현재 Docker DB 상태도 백업**합니다.

```bash
mkdir -p backups/day07

docker compose exec -T db sh -c \
  'pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc' \
  > backups/day07/docker_day07.dump

ls -lh backups/day07/docker_day07.dump
pg_restore -l backups/day07/docker_day07.dump | head -12
```

이 파일은 **백업일 뿐 현재 데이터베이스를 자동으로 복구하는 프로그램은 아닙니다.** 프로젝트 외장 백업이나 기관 승인 저장소에 적절히 보관합니다. 원본 DB 백업에는 검사기록이나 학생 실습 내용이 들어 있을 수 있으므로 공개 Git에 올리지 않습니다.

### 실습 22. Raspberry Pi·교과 9/10 외부 추론 연결 범위를 구분합니다

7일차의 기본 FastAPI 포트는 **이 PC에서만 접속 가능한 `127.0.0.1:8000`**입니다. 따라서 교과 13 플랫폼이 정상 작동해도 Raspberry Pi에서 PC의 사설 IP로 접속하려면 추가 네트워크 설정이 필요합니다.

**Raspberry Pi와 다시 연결해야 하는 경우에만**, 내부망 접속 허용 범위를 확인하고 `.env`의 다음 값 하나를 변경합니다.

```dotenv
# 현재 PC의 허가된 네트워크 인터페이스에서 접속받도록 설정할 때만 변경
PLATFORM_BIND_HOST=0.0.0.0
```

`0.0.0.0`은 PC에서 접근 가능한 모든 인터페이스에 포트가 노출될 수 있는 설정입니다. 인터넷·공용망에 공개하지 말고 **기관의 접근 제어와 방화벽 규칙을 확인한 뒤** 사용해야 합니다. 변경 후 API만 재생성합니다.

```bash
docker compose up -d --force-recreate api
```

Pi의 6일차 `device_config.json`은 여전히 **PC의 실제 IP와 포트 8000**을 바라보아야 합니다. 컨테이너 안의 `db`나 `api` 서비스명을 Pi에서 그대로 사용하는 것은 아닙니다.

```text
Raspberry Pi       → http://PC의_허가된_사설_IP:8000
Docker 내부 FastAPI → db:5432, redis:6379
```

Pi에서 연결이 되지 않으면 Windows 방화벽·Docker Desktop 포트 바인딩·WSL 네트워크 방식·기관 Wi-Fi 단말 격리를 확인합니다. Pi와 PC가 같은 네트워크라고 해서 모든 환경에서 자동으로 포트가 열리는 것은 아닙니다.

4일차 교과 9 OCR 및 교과 10 Pose Bridge도 **PC의 별도 서비스**입니다. `host.docker.internal` 접속은 운영체제와 Listen 주소에 따라 달라집니다. 모델 서버가 실행되고 있지 않다면 `/live`에서 연결 오류가 보이는 것이 정상이며, **오늘의 Docker Compose·PostgreSQL·Redis·Celery 성공과 구분**합니다.

---

## PART 7. 자동 테스트와 오류 해결

### 실습 23. 7일차 상태 API 자동 테스트를 작성합니다

브라우저에서 연결 결과를 확인하는 것만으로는 상태 API가 잘못된 정보를 전달하는지 알기 어렵습니다. 새 API가 정상·장애 상태를 구분하고, 비밀번호를 응답에 포함하지 않는지 Pytest로 검사합니다.

오늘의 테스트는 외부 Docker나 DB를 직접 실행하지 않고, **상태 확인 함수의 반환값을 테스트 조건으로 바꾸어 API 반응**을 확인합니다. 실제 서비스 연결은 **실습 17·18**에서 따로 검증했습니다.

**의사코드**

```text
DB·Redis·Worker가 모두 정상이라고 가정 → status=ok
       ↓
DB만 실패하도록 바꿈 → status=degraded
       ↓
Redis만 실패하도록 바꿈 → status=degraded
       ↓
Worker만 실패하도록 바꿈 → status=degraded
       ↓
응답 JSON에 비밀번호·접속주소가 없는지 확인
       ↓
/containers 화면과 기존 화면 이동 링크 확인
```

**파일: `tests/test_day07.py`**

```python
"""7일차: Docker 서버 없이도 새 상태 API와 화면을 검증합니다."""
from fastapi import FastAPI
from fastapi.testclient import TestClient
import app.container_api as module


def make_client(monkeypatch, db=True, redis=True, worker=True):
    monkeypatch.setattr(module, "probe_postgresql", lambda: db)
    monkeypatch.setattr(module, "probe_redis", lambda: redis)
    monkeypatch.setattr(module, "probe_worker", lambda: worker)
    app = FastAPI()
    app.include_router(module.router)
    return TestClient(app)


def test_services_ready(monkeypatch):
    response = make_client(monkeypatch).get("/api/v1/containers/status")
    assert response.status_code == 200
    result = response.json()
    assert result["status"] == "ok"
    assert all(result["services"].values())


def test_postgresql_unavailable(monkeypatch):
    result = make_client(monkeypatch, db=False).get("/api/v1/containers/status").json()
    assert result["status"] == "degraded"
    assert result["services"]["postgresql"] is False


def test_redis_unavailable(monkeypatch):
    result = make_client(monkeypatch, redis=False).get("/api/v1/containers/status").json()
    assert result["status"] == "degraded"
    assert result["services"]["redis"] is False


def test_worker_unavailable(monkeypatch):
    result = make_client(monkeypatch, worker=False).get("/api/v1/containers/status").json()
    assert result["status"] == "degraded"
    assert result["services"]["celery_worker_responding"] is False


def test_no_password_or_redis_url_in_payload(monkeypatch):
    response = make_client(monkeypatch).get("/api/v1/containers/status")
    assert "password" not in response.text.lower()
    assert "redis://" not in response.text.lower()


def test_status_note_distinguishes_worker_from_job_success(monkeypatch):
    result = make_client(monkeypatch).get("/api/v1/containers/status").json()
    assert "작업 결과" in result["note"]


def test_web_page(monkeypatch):
    response = make_client(monkeypatch).get("/containers")
    assert response.status_code == 200
    assert "Docker Compose" in response.text
    assert "Celery" in response.text


def test_page_has_pipeline_and_pi_links(monkeypatch):
    response = make_client(monkeypatch).get("/containers")
    assert 'href="/pipeline"' in response.text
    assert 'href="/edge-device"' in response.text
```

먼저 **기존 PC `.venv` 환경**에서 자동 테스트를 실행합니다. 테스트는 외부 서버에 연결하지 않으므로 Docker가 없어도 새 상태 API를 검증할 수 있습니다.

```bash
python -m pytest -q tests/test_day07.py
```

같은 코드를 컨테이너 이미지로도 확인하려면, 테스트 파일이 이미지 빌드 이후에 만들어졌으므로 **파일 하나만 읽기 전용으로 마운트**해서 실행합니다.

```bash
docker compose run --rm \
  -v "$(pwd)/tests/test_day07.py:/app/tests/test_day07.py:ro" \
  api python -m pytest -q tests/test_day07.py
```

정상일 때 이번 신규 테스트 **8개가 통과**합니다.

```text
........                                                                 [100%]
8 passed
```

앞선 일차의 테스트 코드와 모델 준비 상태에 따라 기존 전체 테스트 결과는 달라질 수 있습니다. 오늘은 먼저 새 테스트 8개를 확인한 뒤, 1~6일차의 개별 테스트도 실제 강의실 환경에 맞게 확인합니다. **이 신규 테스트만으로 Docker 배포 전체가 검증되었다고 해석하지 않습니다.**

### 실습 24. 오류가 발생하면 문제가 시작된 단계를 확인합니다

| 증상 | 먼저 확인할 곳 | 해결 방법 |
|---|---|---|
| `docker: command not found` | Docker 설치 및 WSL 통합 | Docker 환경을 먼저 준비하고 `docker info` 확인 |
| `Cannot connect to the Docker daemon` | Docker 엔진 실행 상태·접근 권한 | Docker Desktop/Engine 상태 확인 |
| `DB_PASSWORD` 또는 `DAY06_DEVICE_API_KEY` 오류 | `.env`의 기존 실제 값 | 기존 일차의 비밀값 확인; 임의로 소스에 직접 쓰지 않기 |
| `port is already allocated` / `8000` | PC에서 실행 중인 Uvicorn | 예전 FastAPI 서버를 `Ctrl+C`로 종료하고 재시도 |
| `api`가 시작하자마자 종료 | `docker compose logs api` | import 누락, Python 패키지, `app/main.py` 확인 |
| `ModuleNotFoundError: onnxruntime` | `requirements.docker.txt`와 이미지 빌드 | 파일에 패키지가 있는지 확인하고 `docker compose build api` 후 재생성 |
| `libgomp.so.1` 없음 | Dockerfile의 `libgomp1` 설치 | Dockerfile 확인 후 이미지 재빌드 |
| `could not translate host name "db"` | Compose 컨테이너의 DB 환경변수 | `DB_HOST=db`, 같은 Compose 네트워크 확인 |
| PostgreSQL 인증 실패 | DB 초기 생성 시 사용한 비밀번호 | `.env` 값과 기존 named volume의 초기 DB 비밀번호가 일치하는지 확인. 재생성만으로 기존 DB 비밀번호가 바뀌지 않음 |
| DB 테이블 없음 | 기존 DB 복원/초기화 누락 | 방법 A의 복원 또는 방법 B의 생성·이관 중 하나 수행 |
| Export 작업이 계속 `PENDING` | Worker 실행·Redis Broker | `docker compose logs worker`, `docker compose exec redis redis-cli ping` 확인 |
| Export `FAILURE` | Celery Worker 로그 | DB 연결, `recorded_result` 기록 유무, Parquet 쓰기 권한 확인 |
| `Permission denied` on Export | PC 바인드 마운트 폴더 권한 | `data/day03/exports/`의 소유자·쓰기 권한 확인 |
| `/live`에서 기존 OCR·Pose 연결 실패 | PC 외부 모델 서버 실행·네트워크 | 기존 교과 서버 상태·주소·Listen 설정 확인 (Compose 기본 연결과 별도) |
| Pi에서 PC API 접속 불가 | `PLATFORM_BIND_HOST`, 사설 IP, 방화벽 | 내부망 접근 승인 후 공개 범위와 네트워크 확인 |
| `docker compose restart` 후 기록이 사라진 것처럼 보임 | 다른 DB에 접속했는지·Volume | Docker PostgreSQL의 테이블과 `docker volume ls` 확인. 즉시 데이터 삭제 명령 실행 금지 |

**기본 점검 순서**

```text
Docker 엔진
    ↓
docker compose config --quiet
    ↓
docker compose ps
    ↓
PostgreSQL·Redis Healthcheck
    ↓
FastAPI 로그 / Swagger
    ↓
Celery Worker 로그
    ↓
Export SUCCESS / 파일 확인
    ↓
재시작 후 DB·파일 보존 확인
```

**문제가 생겼다고 앞의 교과 프로젝트·DB 볼륨을 삭제하지 않습니다.** 오류가 난 서비스의 로그와 설정을 먼저 확인합니다.

---

## PART 8. 프로젝트 구조·실행 방법을 문서로 남깁니다

### 실습 25. Docker 구성과 복구 방법을 별도 문서로 기록합니다

오늘 만든 네 서비스가 어떤 경로로 통신하고, 어떤 데이터가 Docker 볼륨에 남는지 문서로 정리합니다. 다음 8일차 Jetson 실습에서도 이 문서를 참조할 수 있습니다.

**파일: `docs/day07_compose.md`**

````markdown
# 교과 13 · 7일차 Docker Compose 실행 메모

## 기존 프로젝트 연결

- 1~6일차 `subject13_ai_platform` 코드 유지
- FastAPI의 기존 Router 및 PostgreSQL 테이블 유지
- 교과 9 OCR·교과 10 YOLO Pose 실행환경과 Raspberry Pi는 별도 장치/프로세스

## Compose 서비스

```text
PC 브라우저 → FastAPI(api:8000)
                   ├──→ PostgreSQL(db:5432)
                   └──→ Redis(redis:6379) → Celery(worker)
                                               └──→ PostgreSQL 조회
                                               └──→ Bronze/Silver/Gold Export
```

## 데이터 보존 위치

- PostgreSQL: `pg_data` named volume
- Redis: `redis_data` named volume (AOF 사용)
- 3일차 Export: `data/day03/exports/` 바인드 마운트
- 1일차 JSON: `handoff/records/` 읽기 전용
- 5일차 ONNX: `artifacts/day05/` 읽기 전용
- DB 백업: `backups/day07/` (Git 제외)

## 실행 순서

```bash
docker compose config --quiet
docker compose build api
docker compose up -d db redis
# 신규 DB에는 기존 기록을 복원하거나, 테이블을 만들고 JSON을 이관한 뒤 실행
docker compose up -d api worker
docker compose ps
docker compose exec api python -m scripts.18_probe_compose
```

## 웹 주소

- `/` : 기존 공통 JSON
- `/db` : PostgreSQL 조회
- `/pipeline` : Celery Export
- `/edge-device` : Raspberry Pi 결과 조회
- `/containers` : Docker 통합 상태
- `/docs` : Swagger

## 장애 시 확인

```bash
docker compose ps
docker compose logs --tail=60 api
docker compose logs --tail=60 worker
docker compose exec redis redis-cli ping
```

## 데이터 보존 주의

- `docker compose restart` : 컨테이너 재시작
- `docker compose down` : 컨테이너 정리, 기본적으로 named volume은 유지
- `docker compose down -v` : **오늘 사용하지 않음** (DB 데이터 손실 가능)
- `backups/day07/*.dump` : 비공개 원본·운영 기록이므로 외부 Git에 업로드하지 않음

## 다음 일차

8일차에는 Jetson Orin Nano Super의 **ARM64/GPU 컨테이너** 환경을 별도로 구성합니다.
오늘 PC의 `python:3.12-slim` 이미지를 Jetson의 CUDA/TensorRT 이미지로 오해하지 않습니다.
````

### 실습 26. 기존 README와 구조도를 업데이트합니다

1~6일차에 작성한 `README.md`를 삭제하지 않고 맨 아래에 다음 항목을 **추가**합니다.

````markdown
## 7일차 — Docker Compose 통합 플랫폼

중앙 FastAPI·PostgreSQL·Redis·Celery Worker를 Docker Compose로 실행합니다.
기존 1~6일차 코드와 API 경로를 유지하고, `/containers`에서 서비스 상태를 확인합니다.

### 최초 실행

```bash
docker compose config --quiet
docker compose build api
docker compose up -d db redis
# 기존 DB 복원 또는 1일차 JSON 재이관 완료 후
docker compose up -d api worker
docker compose exec api python -m scripts.18_probe_compose
```

### 접속

- 상태 페이지: http://127.0.0.1:8000/containers
- 기존 기록/DB/Export: `/`, `/db`, `/pipeline`

### 데이터 보관

DB/Redis는 named volume에 보존하며, Export는 `data/day03/exports/`에 보존합니다.
`.env`, 모델, 원본 데이터, DB Dump는 Git에 올리지 않습니다.
````

`docs/architecture.md`에도 다음 내용을 **맨 아래에 추가**합니다.

````markdown
## 7일차 — Docker Compose 중앙 플랫폼

기존 FastAPI + PostgreSQL + Redis + Celery를 4개 서비스로 분리하여 컨테이너화했습니다.
`app/main.py`의 1~6일차 API Router와 `inference_events` 테이블을 그대로 사용합니다.
PostgreSQL과 Redis는 내부 네트워크 및 named volume으로 관리하며,
기존 JSON·ONNX·Export 결과는 bind mount로 전달합니다.
교과9·10 추론 Bridge와 Raspberry Pi는 컨테이너 밖에서 실행되는 별도 구성요소입니다.
````

### 실습 27. `.gitignore`를 확인하고 오늘의 소스만 기록합니다

`.env`, 기존 모델·원본 결과뿐 아니라 DB 백업을 실수로 Git에 올리지 않도록 **기존 `.gitignore` 아래에** 다음 내용을 추가합니다.

```gitignore
# 7일차 Docker DB 백업과 로컬 서비스 데이터
backups/day07/
*.dump
```

이미 앞선 일차에 기록한 `.gitignore`의 `.env`, `handoff/records/`, `artifacts/`, `data/day03/exports/` 제외 규칙은 **삭제하지 않습니다.** 필요하면 Git이 실제로 무엇을 추적하는지 다음 명령으로 확인합니다.

```bash
git status --short
git check-ignore .env backups/day07/pre_docker.dump || true
```

**체크포인트:** 비밀번호·장치 키·모델 파일·기존 JSON·DB 백업이 Git Staged 목록에 나타나지 않아야 합니다.

---

# 6. 오늘 작성한 내용의 최종 정리 및 GitHub 업로드

## 6.1. 오늘 무엇을 완성했나요?

1~6일차의 프로그램을 처음부터 다시 작성하지 않고, **기존 중앙 플랫폼 실행환경을 Docker Compose로 묶었습니다.**

```text
교과 9 실제 OCR 결과 ──┐
                      ├── 기존 공통 이벤트·FastAPI ──┐
교과 10 Pose/Action ───┘                            │
                                                   ▼
                                               Docker API
                                                   │
                                    ┌──────────────┼────────────┐
                                    ▼              ▼            ▼
                              PostgreSQL        Redis       웹브라우저
                                    ▲              │
                                    │              ▼
                                    └────── Celery Worker
                                           │
                                           └── JSONL · Parquet · Gold

Raspberry Pi ONNX·GPIO ── HTTP ──▶ 기존 Pi 수신 API (허가된 내부망 연결 시)
```

이 구조에서는 AI 모델 실행환경과 중앙 데이터·서비스 플랫폼의 역할이 구분됩니다. Docker로 중앙 서비스를 묶어 실행했다고 해서 **Jetson GPU나 TensorRT까지 배포한 것은 아닙니다.** Jetson용 ARM64/GPU 컨테이너는 8일차에 별도로 배웁니다.

## 6.2. 오늘의 자가 체크리스트

- [ ] 6일차까지 사용한 프로젝트를 그대로 열었고 기존 파일을 삭제하지 않았습니다.
- [ ] Docker Engine과 Docker Compose v2가 실행되는 것을 확인했습니다.
- [ ] 기존 PostgreSQL 기록을 백업했거나, 1일차 JSON 재이관 경로를 준비했습니다.
- [ ] `requirements.docker.txt`, `Dockerfile`, `.dockerignore`를 작성했습니다.
- [ ] `compose.yaml`에 `api`, `db`, `redis`, `worker` 네 서비스가 정의되어 있습니다.
- [ ] `docker compose config --quiet`가 성공했습니다.
- [ ] Docker 이미지 `subject13-platform:day07`을 빌드했습니다.
- [ ] Docker PostgreSQL에 1~6일차의 필요한 기록을 복원하거나 JSON을 다시 이관했습니다.
- [ ] `docker compose ps`에서 네 서비스의 실행 상태를 확인했습니다.
- [ ] `/containers`에서 DB·Redis·Worker 상태가 표시됩니다.
- [ ] 3일차 Export 작업이 `SUCCESS`이고 JSONL·Parquet·Gold 파일을 확인했습니다.
- [ ] `python -m pytest -q tests/test_day07.py`로 8개 새 테스트가 통과했습니다.
- [ ] 컨테이너를 재시작한 뒤에도 PostgreSQL 기록과 Export 파일이 유지됩니다.
- [ ] README, 구조 문서, 운영 메모를 작성했습니다.
- [ ] 비밀키·모델·백업 자료가 Git Staged 목록에 없는지 확인했습니다.

## 6.3. Git에 7일차 작업을 기록합니다

1~6일차와 동일한 Git 저장소에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git status
```

오늘 새로 작성한 **코드와 설정 문서만** 추가합니다.

```bash
git add Dockerfile compose.yaml .dockerignore requirements.docker.txt \
  app/container_api.py \
  scripts/17_check_day07_inputs.py scripts/18_probe_compose.py \
  web/containers.html tests/test_day07.py docs/day07_compose.md \
  app/main.py README.md docs/architecture.md .gitignore .env.example

git status
```

`git status`에서 `.env`, `backups/day07/`, `handoff/records/`, `artifacts/day05/`, `data/day03/exports/`가 Staged 목록에 없는지 확인한 뒤 커밋합니다.

```bash
git commit -m "교과13 7일차: Docker Compose 통합 플랫폼 및 데이터 보존 검증"
git log -1 --oneline
git status
```

외부 GitHub 사용이 허용되는 실습환경에서 이미 `origin`이 설정되어 있다면 다음 명령으로 올릴 수 있습니다.

```bash
git remote -v
git push origin main
```

외부 GitHub가 금지된 폐쇄망에서는 **로컬 Git 또는 기관이 허용한 내부 GitLab/Gitea**를 사용합니다. 원본 기록·모델·DB 백업·인증 키는 공개하지 않습니다.

## 6.4. 다음 8일차와 어떻게 연결되나요?

7일차에서는 PC 중앙 플랫폼에 Docker를 처음 도입했습니다. 8일차에는 **NVIDIA Jetson Orin Nano Super의 JetPack·GPU·ARM64 환경**을 확인하고, 이미 만든 모델과 추론 서비스를 Jetson용 컨테이너에서 실행하도록 확장합니다.

다음 수업에서도 `subject13_ai_platform`의 공통 JSON·FastAPI·PostgreSQL 구조는 유지합니다. **오늘 만든 PC Docker 이미지와 Jetson GPU 이미지는 별개의 실행환경**입니다. Jetson에 맞는 NVIDIA 기반 이미지와 Container Runtime, 모델·ONNX를 따로 검증하게 됩니다.

---

**7일차 마무리:** 오늘은 1~6일차에서 이미 만든 기능을 재사용하여, **Docker Compose로 API·PostgreSQL·Redis·Celery를 통합 실행하고, 비동기 작업·기존 기록·재시작 후 데이터 보존까지 확인**했습니다.
