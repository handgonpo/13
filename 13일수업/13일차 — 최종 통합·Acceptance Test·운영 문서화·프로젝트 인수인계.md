## 1. 오늘의 목표

지금까지 교과 13에서 같은 `subject13_ai_platform` 프로젝트에 기능을 하나씩 추가했습니다. 1일차에는 교과 9의 OCR 결과와 교과 10의 동작인식 결과를 공통 JSON으로 연결했고, 2~3일차에는 PostgreSQL과 Redis·Celery를 추가했습니다. 4일차에는 실제 AI 추론을 연결했으며, 5~9일차에는 Raspberry Pi와 Jetson을 위한 모델과 배포를 진행했습니다. 10일차에는 모니터링을, 11~12일차에는 AWS GPU 배포와 Git CI/CD·Rollback을 구축했습니다.

오늘은 **새로운 AI 모델이나 별도의 플랫폼을 만들지 않습니다.** 지금까지 만든 시스템이 실제로 동작하는지 검사하고, 다른 사람이 같은 서비스를 실행·점검·복구할 수 있도록 결과와 방법을 정리합니다.

수업을 마치면 다음 작업을 직접 수행할 수 있어야 합니다.

- [ ] 12일차 프로젝트와 배포 중인 Git 커밋이 일치하는지 확인할 수 있습니다.
- [ ] FastAPI·PostgreSQL·Redis·Celery·Cloud GPU 상태를 검사하고 결과를 `PASS/FAIL/BLOCKED`로 기록할 수 있습니다.
- [ ] 허가된 실제 이미지를 AWS GPU에 입력하여 Pose 결과와 PostgreSQL 저장 기록을 확인할 수 있습니다.
- [ ] 비동기 Export를 실행한 후 `SUCCESS` 및 Bronze·Silver·Gold 파일 생성을 확인할 수 있습니다.
- [ ] Raspberry Pi·Jetson·교과 9/10 서비스는 각각 자신의 기능 범위에서 직접 검증할 수 있습니다.
- [ ] PostgreSQL 백업을 **운영 DB가 아닌 임시 DB**로 복원하고 원본과 복원 건수를 비교할 수 있습니다.
- [ ] Celery Worker 장애와 재시작 후 상태를 확인하고 Rollback 검증의 범위를 정확히 설명할 수 있습니다.
- [ ] Acceptance 결과·운영 Runbook·인수인계 문서를 정리할 수 있습니다.

**오늘의 완료 기준:** 자동 테스트 통과와 실제 장비/서비스 검증을 구별해 기록합니다. **확인하지 못한 기능은 PASS가 아니라 `BLOCKED` 또는 `NOT_RUN`**으로 남깁니다. 필수 검사 중 하나라도 실패하면 전체 인수검사를 완료했다고 적지 않습니다.

### 4시간 진행 기준

| 구간 | 권장 시간 | 오늘 진행할 작업 |
|---|---:|---|
| PART 1~2 | 55분 | 기존 프로젝트 점검, Acceptance 프로그램·테스트 작성 |
| PART 3 | 35분 | 결과 문서·Runbook·Git 기록, 검증 버전 배포 |
| PART 4 | 65분 | AWS·DB·Cloud GPU·Celery·Grafana 실측 |
| PART 5~6 | 85분 | 복원·장애·Edge 장치 확인, 최종 결과 기록·인수인계 |
| **합계** | **240분** | **4시간** |

시간은 한 수업 기준의 진행 계획입니다. AWS/Jetson/Pi가 이미 배포·연결되어 있다는 전제에서 진행하며, 최초 장비 설치나 모델 재학습 시간은 포함하지 않습니다.

---

## 2. 오늘 수업의 전체 흐름

![[Pasted image 20261018133637.png]]

```text
[12일차에 완성한 동일한 subject13_ai_platform]
          │
          ├─ FastAPI / PostgreSQL / Redis / Celery
          ├─ 교과9 OCR / 교과10 Pose·Action
          ├─ Raspberry Pi TinyCNN ONNX + GPIO
          ├─ Jetson TensorRT Pose + 카메라
          ├─ AWS EC2 GPU Pose / Prometheus / Grafana
          └─ GitHub Actions / AWS SSM / Rollback
          ↓
  기존 서비스·코드·배포 SHA 확인
          ↓
  13일차 Acceptance 프로그램 작성
          ↓
  자동 테스트 → Git CI → AWS 배포 버전 확정
          ↓
  [실제 서비스 점검]
    ├─ FastAPI·DB·Worker·Redis
    ├─ 실제 이미지 → AWS Pose → DB 저장
    ├─ Celery Export → Bronze/Silver/Gold
    ├─ Jetson·Raspberry Pi·교과9/10
    └─ Prometheus·Grafana
          ↓
  [데이터/장애 점검]
    ├─ PostgreSQL → 백업 → 임시 DB 복원 → 건수 대조
    ├─ Celery Worker 일시 중단 → 복구
    └─ 이전 12일차 Rollback 테스트 재검증
          ↓
  PASS / FAIL / BLOCKED / NOT_RUN 판정
          ↓
  Acceptance Report · Runbook · 인수인계 기록
          ↓
  교과 14 제조 AI / 교과 15 산업안전 AI에 재사용
```

**중요:** 13일차에는 정상으로 동작하던 프로젝트를 삭제하거나 처음부터 설치하지 않습니다. 기존 PostgreSQL 볼륨, AI 모델, `.env`, 원본 이미지·영상은 유지합니다.

---

## 3. 실습 전에 알아야 할 핵심

### 3.1. Acceptance Test는 무엇인가요?

`pytest` 테스트가 통과하는 것은 **코드의 특정 기능이 예상대로 동작한다는 근거**입니다. 반면 Acceptance Test(최종 인수검사)는 실행 중인 시스템이 요구사항을 실제 환경에서 충족하는지 확인하는 절차입니다.

| 구분 | 검사 대상 | 확인할 결과 |
|---|---|---|
| Python 단위·회귀 테스트 | 함수, JSON, API 응답, 검증 로직 | 테스트 PASS |
| 통합검사 | 여러 서비스 연결 | API → GPU → DB 등 실제 데이터 이동 |
| 장비검사 | Raspberry Pi·Jetson·실제 카메라·GPIO | 장비에서 직접 촬영·추론·제어 |
| 운영검사 | 백업·복원·장애·재시작·모니터링 | 복구 후 이전 기능 유지 |
| 인수인계 | 실행방법·설정·제약·장애 절차 | 다른 사람이 문서대로 실행·복구 가능 |

오늘 사용하는 상태는 네 가지입니다.

- **PASS:** 해당 실제 검증을 수행하여 조건을 만족했습니다.
- **FAIL:** 실제 실행했지만 조건을 만족하지 못했습니다.
- **BLOCKED:** 필요한 모델·장비·권한·네트워크가 없어 확인하지 못했습니다.
- **NOT_RUN:** 아직 실행하지 않았습니다. 실패와 동일한 뜻은 아니지만 합격 처리할 수 없습니다.

### 3.2. 교과 9·10·5·6·8·9·11의 AI는 서로 다른 일을 합니다

| 기능 | 실제 입력 | 판정 및 주의 사항 |
|---|---|---|
| 교과 9 OCR 검사 | 기존 실습 제품 이미지 | SN 문자열 검증의 `OK/NG` |
| 교과 10 Action Recognition | IP Webcam·영상 → Pose → 38-D·Sliding Window·RF | 예측 동작과 기대 동작 비교, `UNKNOWN` 유지 |
| 5~6일차 TinyCNN | `normal`/`warning_red` 카드 이미지 | Raspberry Pi 교육용 2클래스 분류와 LED 제어 |
| 8~9일차 Jetson | 영상/JPG → YOLO11n-Pose → TensorRT | 사람 BBox·COCO17 Keypoint, **동작 판정과 다름** |
| 11~12일차 AWS GPU | JPG/PNG → YOLO11n-Pose CUDA | Pose 검출만 수행하므로 공통 이벤트 판정은 **`UNKNOWN`** |

AWS GPU에서 사람의 Skeleton이 나왔다고 해서 산업안전 동작인식까지 완성된 것은 아닙니다. 교과 15에서 동작 의미와 위험 판정 기준을 확장합니다.

### 3.3. 오늘 반드시 필요한 기존 결과물

| 이전 일차 | 필요 자료·기능 | 없거나 작동하지 않을 때 |
|---|---|---|
| 1~2일차 | `app/schemas.py`, `app/models.py`, PostgreSQL `inference_events` | 해당 부분은 BLOCKED. 이전 수업의 실제 소스·DB를 먼저 복구 |
| 3일차 | `app/celery_app.py`, `app/worker_tasks.py`, Redis, Export API | Worker 결과까지 확인할 수 없으면 BLOCKED |
| 4일차 | 교과 9 OCR·교과 10 실시간 추론 환경 | 4일차 실습 데이터·모델·기존 서비스 연결 복구 |
| 5~6일차 | TinyCNN ONNX·Raspberry Pi·USB 카메라·LED | Edge 검사는 BLOCKED, 임의 예제 추론으로 대체하지 않음 |
| 8~9일차 | Jetson Orin Nano Super·TensorRT 엔진·카메라 | Jetson 검사는 BLOCKED. PC CPU 테스트로 대체하지 않음 |
| 10일차 | `monitoring/`, Prometheus·Grafana | 실제 수집·대시보드 확인 불가이면 BLOCKED |
| 11~12일차 | EC2 GPU, 4개 Compose 파일 + `compose.deploy.yaml`, `.env`, 실제 Pose 가중치, Git/SSM | AWS 인수검사는 BLOCKED. 실제 자원 복구 후 재검증 |

앞 수업을 결석해 결과물을 완성하지 못한 경우에는 **완성된 해당 교과의 실제 산출물**(예: 모델 가중치·설정·DB 백업·실행 코드)을 기관에서 제공받아야 합니다. 임의로 생성한 예측값이나 빈 DB만으로 실제 인수검사를 통과할 수 없습니다. 이전 프로젝트 전체 ZIP을 새로 내려받는 단계는 진행하지 않습니다.

### 3.4. 어디에서 명령어를 실행할까요?

문서에서 실행 위치를 다음과 같이 구별합니다.

- **[개발 PC]**: 이전부터 사용한 WSL Ubuntu의 `~/ai_vision/subject13_ai_platform`.
- **[AWS EC2]**: 11~12일차에 배포한 **기존** EC2의 동일 프로젝트 경로.
- **[Jetson]**: 실제 Jetson 장치 터미널.
- **[Raspberry Pi]**: 실제 Pi 장치 터미널.

PC와 AWS의 `127.0.0.1`은 서로 다른 컴퓨터를 가리킵니다. SSH 터널을 사용하면 PC의 `127.0.0.1:8000`을 EC2로 전달할 수 있으므로, **PC의 기존 8000 서비스와 동시에 같은 포트를 사용할 수 없습니다.** 각 점검의 실행 위치와 서버를 반드시 확인하세요.

### 3.5. 운영 데이터는 어떻게 보호하나요?

이전 회차부터 사용한 DB와 Docker 볼륨은 삭제하지 않습니다. **`docker compose down -v` 또는 `docker volume rm`을 실행하지 않습니다.** 새 버전 설치·검증을 위해 운영 DB를 `DROP`, `TRUNCATE` 하거나 빈 데이터로 초기화하지 않습니다.

13일차 스크립트에서 생성하는 세부 증빙과 실제 개인정보·원본 영상·백업·비밀키는 Git에 올리지 않습니다. Git에는 공개가 허용된 검사 **코드와 빈 문서 양식**만 기록합니다.

---

## 4. 오늘 사용할 기술 스택

| 기술 | 오늘 사용하는 기능 |
|---|---|
| Python 표준 라이브러리(`json`, `urllib`, `subprocess`) | 기존 API 확인, Acceptance JSON 생성, 백업·복원 실행 |
| FastAPI | 1~12일차 API가 실제로 응답하는지 검사 |
| PostgreSQL (`pg_dump`, `pg_restore`) | 데이터 백업·임시 DB 복원·건수 비교 |
| Redis·Celery | Worker·비동기 데이터 Export 성공 확인 |
| Docker Compose | API·DB·Redis·Worker·Prometheus·Grafana·Cloud GPU 실행상태 확인 |
| NVIDIA Jetson / Raspberry Pi | 이미 배포된 장치의 모델·카메라·GPIO 확인 |
| Prometheus / Grafana | 실제 서비스 상태·성능과 장애 발생 여부 확인 |
| Git · GitHub Actions · AWS SSM | 검증 버전 고정 및 배포·Rollback 결과 확인 |
| Pytest | 신규 검사 코드의 판정·안전 조건 자동 검증 |

오늘은 새로운 Dockerfile이나 GPU 모델을 다시 만들지 않습니다. **12일차에 정상적으로 배포한 환경을 점검하는 날**입니다.

---

# 5. 처음부터 끝까지 따라 하는 상세 실습

## PART 1. 12일차 프로젝트와 준비 상태를 확인합니다

### 실습 1. 기존 교과 13 프로젝트를 다시 엽니다

**이번 실습에서는 무엇을 하나요?** 12일차 소스가 남아 있는 폴더로 이동하고 Python·Git 환경을 확인합니다. **13일차라는 이유로 새 프로젝트를 만들지 않습니다.**

**[개발 PC]** WSL 터미널:

```bash
cd ~/ai_vision/subject13_ai_platform
pwd
source .venv/bin/activate
python --version
git status --short
git log -1 --oneline
code .
```

`pwd`에 `subject13_ai_platform`이 표시되고 12일차 최종 커밋이 확인되어야 합니다. 가상환경이 없다면 이전 회차의 `requirements.txt`로 같은 프로젝트의 `.venv`를 복구하세요.

### 실습 2. 이전 실습 파일과 실제 실행환경을 확인합니다

**[개발 PC]** 다음 명령으로 12일차 배포 파일과 데이터 구조를 확인합니다.

```bash
ls -lh scripts/12_dc.sh scripts/12_release_check.py
ls -lh compose.yaml compose.monitoring.yaml compose.cloud.yaml
ls -lh compose.cloud.monitoring.yaml compose.deploy.yaml
ls -lh .github/workflows/day12-ci-cd.yml
ls -lh app/models.py app/cloud_api.py app/version_api.py
ls -lh monitoring/prometheus.cloud.yml
```

**[AWS EC2]** 서버에 접속하여 다음도 확인합니다. EC2 접속 명령과 주소는 11일차에 사용한 승인된 값을 그대로 사용합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git rev-parse HEAD
bash scripts/12_dc.sh ps
curl -fsS http://127.0.0.1:8000/health
curl -fsS http://127.0.0.1:8000/api/v1/cloud/health
```

여기서 `.env` 파일 내용이나 키를 화면에 출력하지 마세요. `bash scripts/12_dc.sh ps`에는 **`db`, `redis`, `api`, `worker`, `prometheus`, `grafana`, `cloud-gpu`**가 표시되어야 하며, 각 서비스의 정상 상태를 함께 확인합니다.

`cloud-gpu`의 Health가 비정상이라면 다음 코드를 작성하기 전에 11~12일차의 Docker 로그와 모델 파일을 확인하세요. 장비가 없는 경우에는 점검 결과를 `BLOCKED`로 기록합니다.

### 실습 3. 13일차에 추가할 폴더를 만듭니다

**[개발 PC]** 프로젝트 최상위에서 실행합니다.

```bash
mkdir -p docs/day13/evidence
mkdir -p scripts tests
```

이번에 추가하는 파일은 다음과 같습니다.

```text
subject13_ai_platform/
├── scripts/
│   ├── 12_dc.sh                         (12일차 유지)
│   ├── 12_release_check.py              (12일차 유지)
│   ├── 13_acceptance.py                 (오늘 추가)
│   └── 13_db_restore_drill.py           (오늘 추가)
├── tests/
│   ├── test_day12_pipeline.py           (12일차 유지)
│   └── test_day13.py                    (오늘 추가)
├── docs/
│   └── day13/
│       ├── ACCEPTANCE_REPORT_TEMPLATE.md  (오늘 추가)
│       ├── RUNBOOK.md                    (오늘 추가)
│       ├── HANDOVER.md                   (오늘 추가)
│       └── evidence/                     (실행 결과, Git 제외)
├── compose*.yaml                        (기존 유지)
├── app/                                 (기존 유지)
└── .gitignore                           (기존 내용에 한 줄만 추가)
```

**지금은 새 폴더만 만들고**, 다음 실습에서 코드를 직접 작성합니다.

---

## PART 2. Acceptance 자동 점검 코드를 작성합니다

### 실습 4. API 상태를 실제 응답으로 검사하는 코드를 만듭니다

**이번 코드는 무엇을 하나요?** 12일차까지 만들어 둔 API를 직접 호출하여 필수 서비스 상태를 검사합니다. 확인 결과는 `PASS`, `FAIL`, `BLOCKED`로 구분하고 JSON 파일로 남깁니다.

**의사코드**

```text
현재 EC2에 배포한 40자리 Git SHA 확인
          ↓
FastAPI /health 호출
          ↓
배포 버전 API와 SHA 비교
          ↓
PostgreSQL / Redis / Celery 연결 확인
          ↓
Cloud GPU CUDA 상태와 저장 기록 확인
          ↓
Prometheus /metrics 확인
          ↓
Jetson·Pi 결과 확인(실제 장비가 있는 경우 필수로 지정)
          ↓
항목별 PASS / FAIL / BLOCKED 결정
          ↓
읽기 전용 결과 JSON으로 저장
```

다음 코드에는 **새 AI 추론 코드가 없습니다.** 기존에 만든 실제 서비스의 준비 상태를 점검하는 용도입니다. `Cloud GPU 저장 기록 있음`은 **과거에 생성된 결과 존재 여부**만 확인하므로, 새로운 이미지를 넣어 실제 추론하는 검사는 뒤의 실습에서 별도로 진행합니다.

**파일: `scripts/13_acceptance.py`**

```python
"""13일차: 현재 실행 중인 플랫폼의 읽기 전용 Acceptance 점검.

실제 모델 추론, 백업 복구, 네트워크 장애와 Rollback 성공을 대신 검증하지 않습니다.
모든 요청은 기본적으로 같은 컴퓨터의 localhost API에만 보냅니다.
"""
import argparse
import json
import re
from datetime import datetime, timezone
from pathlib import Path
from urllib.error import HTTPError, URLError
from urllib.parse import urlsplit
from urllib.request import urlopen

SHA_PATTERN = re.compile(r"^[0-9a-f]{40}$")


def validate_base_url(url: str) -> str:
    parts = urlsplit(url)
    if (parts.scheme != "http" or parts.hostname not in ("127.0.0.1", "localhost")
            or parts.username or parts.password or parts.query or parts.fragment or parts.path not in ("", "/")
            or parts.port != 8000):
        raise ValueError("API 주소는 http://127.0.0.1:8000 또는 http://localhost:8000 이어야 합니다.")
    return url.rstrip("/")


def fetch_json(url: str):
    with urlopen(url, timeout=6) as response:
        if response.status != 200:
            raise RuntimeError(f"HTTP {response.status}")
        return json.load(response)


def fetch_metrics(url: str) -> str:
    with urlopen(url, timeout=6) as response:
        if response.status != 200:
            raise RuntimeError(f"HTTP {response.status}")
        return response.read(1_000_000).decode("utf-8", "replace")


def evaluate(code: str, label: str, required: bool, assertion, *, blocked_if=None):
    """assertion은 성공 여부(bool)를 반환합니다. 미확인 항목은 PASS로 표시하지 않습니다."""
    try:
        observed = assertion()
        if observed is True:
            return {"id": code, "name": label, "required": required, "status": "PASS", "note": "실제 응답 검사 통과"}
        status = "BLOCKED" if blocked_if is not None and blocked_if() else "FAIL"
        return {"id": code, "name": label, "required": required, "status": status,
                "note": "필요 조건 미충족 또는 예상과 다른 응답"}
    except (HTTPError, URLError, TimeoutError, OSError, RuntimeError, ValueError, KeyError,
            TypeError, json.JSONDecodeError) as error:
        return {"id": code, "name": label, "required": required, "status": "BLOCKED",
                "note": f"연결/응답 확인 불가: {type(error).__name__}"}


def run_checks(base_url: str, expected_sha: str, require_jetson: bool, require_pi: bool):
    base = validate_base_url(base_url)
    if not SHA_PATTERN.fullmatch(expected_sha):
        raise ValueError("--expected-sha에 git rev-parse HEAD의 40자리 소문자 SHA를 입력하세요.")
    results = []

    def add(code, label, required, path, validator):
        results.append(evaluate(code, label, required,
                                lambda: bool(validator(fetch_json(base + path)))))

    add("AT-01", "FastAPI 기본 상태", True, "/health",
        lambda d: isinstance(d, dict) and d.get("status") == "ok")
    add("AT-02", "배포 SHA 일치", True, "/api/v1/deploy/version",
        lambda d: isinstance(d, dict) and d.get("commit") == expected_sha)
    add("AT-03", "PostgreSQL SQLAlchemy 연결", True, "/api/v1/db/health",
        lambda d: isinstance(d, dict) and d.get("database") == "connected")
    add("AT-04", "이전 AI 기록 DB 조회", True, "/api/v1/db/summary",
        lambda d: isinstance(d, dict) and isinstance(d.get("total"), int) and d["total"] >= 1)
    add("AT-05", "PostgreSQL·Redis·Celery 응답", True, "/api/v1/containers/status",
        lambda d: isinstance(d, dict) and d.get("status") == "ok" and
        all(d.get("services", {}).get(x) is True for x in
            ("postgresql", "redis", "celery_worker_responding")))
    add("AT-06", "비동기 처리 연결 준비", True, "/api/v1/pipeline/health",
        lambda d: isinstance(d, dict) and d.get("postgresql") is True and d.get("redis") is True)
    add("AT-07", "AWS CUDA Pose 서비스 연결", True, "/api/v1/cloud/health",
        lambda d: isinstance(d, dict) and d.get("connected") is True and d.get("runtime") == "pytorch_cuda")
    add("AT-08", "Cloud GPU 과거 저장 기록 확인", True, "/api/v1/cloud/events?limit=20",
        lambda d: isinstance(d, list) and any(
            isinstance(x, dict) and str(x.get("event_id", "")).startswith("cloud:")
            and x.get("source_mode") == "live_inference" for x in d))
    results.append(evaluate("AT-09", "FastAPI Prometheus 지표", True,
                            lambda: "platform_http_requests_total" in fetch_metrics(base + "/metrics")))
    add("AT-10", "Jetson GPU 서비스 연결", require_jetson, "/api/v1/jetson/health",
        lambda d: isinstance(d, dict) and d.get("connected") is True and
        d.get("runtime") == "tensorrt")
    add("AT-11", "Raspberry Pi 기존 저장 기록", require_pi,
        "/api/v1/edge-device/events?limit=10",
        lambda d: isinstance(d, list) and len(d) >= 1 and any(
            str(x.get("event_id", "")).startswith("pi:") or
            x.get("details", {}).get("device_kind") == "raspberry_pi" for x in d
            if isinstance(x, dict)))

    overall = "PASS" if all(x["status"] == "PASS" for x in results if x["required"]) else "FAIL"
    return {"schema_version": 1, "checked_at_utc": datetime.now(timezone.utc).isoformat(),
            "expected_sha": expected_sha, "api": base, "overall": overall,
            "notice": "읽기 전용 자동 점검입니다. 새 이미지 추론·Celery 실제 완료·DB 복원·Rollback은 별도 실습 결과가 필요합니다.",
            "results": results}


def main() -> None:
    parser = argparse.ArgumentParser(description="교과13 13일차 읽기 전용 인수 점검")
    parser.add_argument("--base-url", default="http://127.0.0.1:8000")
    parser.add_argument("--expected-sha", required=True)
    parser.add_argument("--require-jetson", action="store_true")
    parser.add_argument("--require-pi", action="store_true")
    parser.add_argument("--output", default="docs/day13/evidence/acceptance.json")
    args = parser.parse_args()
    report = run_checks(args.base_url, args.expected_sha, args.require_jetson, args.require_pi)
    target = Path(args.output)
    target.parent.mkdir(parents=True, exist_ok=True)
    target.write_text(json.dumps(report, ensure_ascii=False, indent=2) + "\n", encoding="utf-8")
    for result in report["results"]:
        print(f'{result["id"]} {result["status"]:7} {result["name"]}')
    print(f'필수 항목 종합: {report["overall"]} | 상세 기록: {target}')
    if report["overall"] != "PASS":
        raise SystemExit(1)


if __name__ == "__main__":
    main()
```

**코드 이해하기**

`run_checks()`는 1~12일차에 만들었던 실제 API 경로를 이용합니다. `/api/v1/deploy/version`은 EC2에서 실행하는 Git SHA와 같아야 하고, `/api/v1/db/health`는 실제 DB 연결을 확인합니다. `/api/v1/containers/status`에서는 DB·Redis·Celery Worker 응답을 각각 검증합니다.

`AT-08`은 이미 보관된 `cloud:` 이벤트를 확인합니다. **새 이미지의 추론 완료를 뜻하지 않습니다.** `--require-jetson`과 `--require-pi`는 해당 장치까지 이번 검사에서 필수로 요구할 때 지정합니다. 장비를 검증하지 못했는데 해당 옵션을 빼고 자동검사만 PASS가 나와도 **전체 13일차 통합 시스템이 PASS가 된 것은 아닙니다.**

보안을 위해 프로그램은 `http://127.0.0.1:8000` 또는 `http://localhost:8000`만 허용합니다. AWS EC2에서는 EC2 자신의 루프백 주소, 개발 PC에서는 해당 포트로 열린 SSH 터널을 사용합니다.

### 실습 5. Acceptance 코드의 판정 규칙을 자동으로 검사합니다

**이번 코드는 무엇을 하나요?** API가 실제로 실패했을 때 검사 프로그램이 거짓 PASS를 반환하지 않는지 확인합니다. `fetch_json()` 부분은 테스트 중에는 준비한 응답으로 바꾸므로 **실제 GPU 장비 없이도 검사 로직 자체**를 테스트할 수 있습니다.

**의사코드**

```text
올바른 상태 응답 → PASS
Cloud GPU 연결 실패 → FAIL
Git SHA 불일치 → FAIL
Jetson이 필수인데 연결 실패 → FAIL
Git에서 위험한 주소 입력 → 검사 거부
운영 DB와 임시 DB 이름 혼동 → 검사 거부
```

**파일: `tests/test_day13.py`**

```python
"""13일차: 장비가 없어도 검사 프로그램의 판정·안전 수칙을 검증합니다.

이 테스트 통과는 운영 장비 Acceptance 합격을 뜻하지 않습니다.
"""
import importlib.util
import subprocess
from pathlib import Path
from unittest.mock import patch

import pytest

ROOT = Path(__file__).resolve().parents[1]


def load(name):
    path = ROOT / "scripts" / name
    spec = importlib.util.spec_from_file_location(name.replace(".", "_"), path)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    return module


acceptance = load("13_acceptance.py")
drill = load("13_db_restore_drill.py")
SHA = "a" * 40


@pytest.fixture
def fake_service():
    items = {
        "/health": {"status": "ok"},
        "/api/v1/deploy/version": {"commit": SHA},
        "/api/v1/db/health": {"database": "connected"},
        "/api/v1/db/summary": {"total": 12},
        "/api/v1/containers/status": {"status": "ok", "services": {
            "postgresql": True, "redis": True, "celery_worker_responding": True}},
        "/api/v1/pipeline/health": {"postgresql": True, "redis": True},
        "/api/v1/cloud/health": {"connected": True, "runtime": "pytorch_cuda"},
        "/api/v1/cloud/events?limit=20": [{"event_id": "cloud:abc", "source_mode": "live_inference"}],
        "/api/v1/jetson/health": {"connected": True, "runtime": "tensorrt"},
        "/api/v1/edge-device/events?limit=10": [{"event_id": "pi:abc", "details": {"device_kind": "raspberry_pi"}}],
    }
    with patch.object(acceptance, "fetch_json", side_effect=lambda url: items[url.removeprefix("http://127.0.0.1:8000")]), \
         patch.object(acceptance, "fetch_metrics", return_value="platform_http_requests_total{status=\"200\"} 3"):
        yield items


def test_acceptance_full_pass_with_devices(fake_service):
    result = acceptance.run_checks("http://127.0.0.1:8000", SHA, True, True)
    assert result["overall"] == "PASS"
    assert len(result["results"]) == 11
    assert all(check["status"] == "PASS" for check in result["results"])


def test_required_gpu_missing_makes_final_fail(fake_service):
    fake_service["/api/v1/cloud/health"] = {"connected": False, "runtime": None}
    result = acceptance.run_checks("http://127.0.0.1:8000", SHA, False, False)
    assert result["overall"] == "FAIL"
    assert result["results"][6]["status"] == "FAIL"


def test_wrong_release_sha_fails(fake_service):
    fake_service["/api/v1/deploy/version"] = {"commit": "b" * 40}
    result = acceptance.run_checks("http://127.0.0.1:8000", SHA, False, False)
    assert result["overall"] == "FAIL"


def test_missing_gpu_event_not_accepted_as_new_inference(fake_service):
    fake_service["/api/v1/cloud/events?limit=20"] = []
    result = acceptance.run_checks("http://127.0.0.1:8000", SHA, False, False)
    assert result["results"][7]["status"] == "FAIL"


def test_jetson_optional_failure_does_not_mask_optional_status(fake_service):
    fake_service["/api/v1/jetson/health"] = {"connected": False}
    result = acceptance.run_checks("http://127.0.0.1:8000", SHA, False, False)
    assert result["overall"] == "PASS"
    assert result["results"][9]["status"] == "FAIL"
    assert result["results"][9]["required"] is False


def test_required_jetson_failure_blocks_acceptance(fake_service):
    fake_service["/api/v1/jetson/health"] = {"connected": False}
    result = acceptance.run_checks("http://127.0.0.1:8000", SHA, True, False)
    assert result["overall"] == "FAIL"


@pytest.mark.parametrize("url", ["http://example.com:8000", "https://127.0.0.1:8000", "http://127.0.0.1:9999",
                                  "http://127.0.0.1:8000/admin", "http://username@127.0.0.1:8000"])
def test_acceptance_only_local_port_8000(url):
    with pytest.raises(ValueError):
        acceptance.validate_base_url(url)


def test_invalid_release_sha_rejected():
    with pytest.raises(ValueError):
        acceptance.run_checks("http://localhost:8000", "short", False, False)


def test_backup_name_validation_prevents_command_injection(tmp_path):
    with pytest.raises(ValueError):
        drill.perform_drill(tmp_path, "'; DROP DATABASE main; --")


def test_backup_empty_original_stops_before_backup(tmp_path):
    with patch.object(drill, "count_events", return_value=0):
        with pytest.raises(ValueError, match="1건 이상"):
            drill.perform_drill(tmp_path, "20261017120000")
    assert list(tmp_path.glob("*.dump")) == []


def test_backup_name_is_temporary_db_only():
    assert drill.DRILL_NAME.fullmatch("subject13_drill_20261017120000")
    assert not drill.DRILL_NAME.fullmatch("subject13_platform")
    assert not drill.DRILL_NAME.fullmatch("subject13_drill_test;rm")


def test_db_restore_script_does_not_delete_production_db():
    source = (ROOT / "scripts/13_db_restore_drill.py").read_text(encoding="utf-8")
    assert 'dropdb -U "$POSTGRES_USER" --if-exists -- "$1"' in source
    assert "if not DRILL_NAME.fullmatch(drill_db)" in source
    assert "DROP DATABASE subject13_platform" not in source


def test_restore_drill_mock_happy_path(tmp_path):
    """모의 pg_dump/pg_restore 경로와 임시 DB 정리가 실제 함수 순서대로 호출되는지 검사."""
    calls = []
    def fake_db(command, database=None, **kwargs):
        calls.append((command, database))
        return subprocess.CompletedProcess([command], 0, stdout="")

    def fake_subprocess(args, **kwargs):
        joined = " ".join(args)
        if "pg_dump" in joined:
            kwargs["stdout"].write(b"PGDMPmock-content")
            return subprocess.CompletedProcess(args, 0)
        if "pg_restore --list" in joined:
            return subprocess.CompletedProcess(args, 0, stdout=b"TABLE public inference_events")
        if "pg_restore" in joined:
            return subprocess.CompletedProcess(args, 0)
        raise AssertionError(f"unexpected {joined}")

    with patch.object(drill, "count_events", side_effect=[11, 11, 11]), \
         patch.object(drill, "docker_db", side_effect=fake_db), \
         patch.object(drill.subprocess, "run", side_effect=fake_subprocess):
        result = drill.perform_drill(tmp_path, "20261017220000")
    assert result["result"] == "PASS"
    assert result["original"] == result["restored"] == result["after"] == 11
    assert "createdb" in calls[0][0] and "dropdb" in calls[-1][0]
    assert Path(result["archive"]).read_bytes().startswith(b"PGDMP")


def test_restore_drill_mock_failure_still_drops_temporary_db(tmp_path):
    calls = []
    def fake_db(command, database=None, **kwargs):
        calls.append(command)
        return subprocess.CompletedProcess([command], 0, stdout="")

    def fake_subprocess(args, **kwargs):
        joined = " ".join(args)
        if "pg_dump" in joined:
            kwargs["stdout"].write(b"PGDMPmock")
            return subprocess.CompletedProcess(args, 0)
        if "pg_restore --list" in joined:
            return subprocess.CompletedProcess(args, 0, stdout=b"TABLE public inference_events")
        if "pg_restore" in joined:
            raise subprocess.CalledProcessError(1, args)
        raise AssertionError(joined)

    with patch.object(drill, "count_events", return_value=11), \
         patch.object(drill, "docker_db", side_effect=fake_db), \
         patch.object(drill.subprocess, "run", side_effect=fake_subprocess):
        with pytest.raises(subprocess.CalledProcessError):
            drill.perform_drill(tmp_path, "20261017220001")
    assert any("createdb" in x for x in calls)
    assert any("dropdb" in x for x in calls)


def test_network_failure_is_blocked_not_success(fake_service):
    from urllib.error import URLError
    with patch.object(acceptance, "fetch_json", side_effect=URLError("no-route")):
        result = acceptance.run_checks("http://127.0.0.1:8000", SHA, False, False)
    assert result["overall"] == "FAIL"
    assert result["results"][0]["status"] == "BLOCKED"
```

**코드 이해하기**

`fake_service`는 **검사 프로그램의 판정 로직을 테스트하기 위한 가짜 API 응답**입니다. 교육용 모델 추론이나 실제 AWS 서버 상태를 대신하지 않습니다. `test_required_gpu_missing_makes_final_fail()`은 CUDA 서비스가 준비되지 않았다면 전체 필수 점검을 실패로 표시해야 한다는 규칙을 확인합니다.

### 실습 6. 새 자동 테스트를 실행합니다

**[개발 PC]** 기존 가상환경에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m pytest -q tests/test_day13.py
```

오늘 제공한 검증 코드 기준으로 19개 테스트가 정의되어 있습니다. 정상 출력의 형식은 다음과 같습니다.

```text
...................                                                      [100%]
19 passed
```

**이 테스트가 통과해도 AWS GPU·Raspberry Pi·Jetson 장비 실측은 아직 완료된 것이 아닙니다.** 기존 전체 회귀 테스트도 함께 실행합니다.

```bash
DATABASE_URL='sqlite+pysqlite:///:memory:' python -m pytest -q \
  tests/test_day08.py tests/test_day09.py \
  tests/test_day10_config.py tests/test_day10_jetson_route.py \
  tests/test_day10_monitoring.py tests/test_day10_telemetry.py \
  tests/test_day11_cloud.py tests/test_day12_pipeline.py \
  tests/test_day12_remote_rollback.py tests/test_day13.py
```

실습용 Python 환경에 이전 회차 테스트가 모두 있다면 정상적으로 수집되어야 합니다. 만약 테스트에 필요한 모델·미설치 패키지가 빠져 실패한다면 어떤 파일이나 환경이 빠졌는지 확인하고 기록합니다. **일부 테스트가 실패해도 출력 건수를 임의로 성공으로 바꾸지 않습니다.**

---

## PART 3. 실제 백업 복원을 위한 코드를 준비합니다

### 실습 7. 운영 DB 대신 임시 DB에 복원하는 코드를 작성합니다

**이번 코드는 무엇을 하나요?** 이전 수업에서는 `pg_dump` 파일을 만들고 보관하는 방법을 배웠습니다. 오늘은 그 백업을 실제로 읽을 수 있는지 확인하고, 별도 테스트 DB에 복원한 뒤 **운영 DB의 이벤트 수와 복원된 이벤트 수가 같은지** 검사합니다.

**의사코드**

```text
운영 PostgreSQL 이벤트 건수 읽기
          ↓
실제 DB의 pg_dump 생성
          ↓
백업 헤더 PGDMP와 pg_restore 목차 검사
          ↓
subject13_drill_날짜시각 임시 DB 생성
          ↓
임시 DB에만 pg_restore 실행
          ↓
원본 건수 = 복원 건수 = 검사 후 원본 건수 ?
          ↓
결과 확인 후 임시 DB 삭제
          ↓
백업은 Git 밖의 운영 폴더에 보존
```

**중요:** 아래 코드는 정상 서비스의 테이블을 `DROP`하거나 원래 DB에 복원하지 않습니다. PostgreSQL 계정에 임시 DB 생성 권한이 있어야 하며, 복원에 필요한 디스크 공간과 작업 시간이 확보되어야 합니다.

**파일: `scripts/13_db_restore_drill.py`**

```python
"""13일차: 운영 DB에는 쓰지 않고, 격리된 임시 DB에 백업을 복원하여 검사.

PostgreSQL 데이터베이스 이름을 삭제/생성하는 작업은 임시 'subject13_drill_' 이름으로만 허용합니다.
DB dump는 Git 밖의 사용자 홈 배포 상태 경로에 저장합니다.
"""
import argparse
import re
import subprocess
from datetime import datetime, timezone
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]
COMPOSE = ["bash", "scripts/12_dc.sh", "exec", "-T", "db", "sh", "-c"]
DRILL_NAME = re.compile(r"^subject13_drill_[0-9]{14}$")


def docker_db(command: str, database: str | None = None, *, stdin=None) -> subprocess.CompletedProcess:
    args = COMPOSE + [command, "sh"]
    if database is not None:
        args.append(database)
    return subprocess.run(args, cwd=ROOT, stdin=stdin, capture_output=stdin is None,
                          text=stdin is None, check=True)


def count_events(database: str | None = None) -> int:
    if database is None:
        command = 'exec psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -At -c "SELECT COUNT(*) FROM inference_events;"'
    else:
        command = 'exec psql -U "$POSTGRES_USER" -d "$1" -At -c "SELECT COUNT(*) FROM inference_events;"'
    result = docker_db(command, database)
    count = result.stdout.strip()
    if not count.isdecimal():
        raise ValueError("SQL COUNT(*) 결과를 읽을 수 없습니다.")
    return int(count)


def perform_drill(backup_dir: Path, stamp: str) -> dict:
    if not re.fullmatch(r"[0-9]{14}", stamp):
        raise ValueError("시각 형식이 잘못되었습니다.")
    drill_db = f"subject13_drill_{stamp}"
    if not DRILL_NAME.fullmatch(drill_db):
        raise ValueError("안전한 임시 DB 이름이 아닙니다.")
    backup_dir.mkdir(parents=True, exist_ok=True)
    archive = backup_dir / f"day13_acceptance_{stamp}.dump"
    if archive.exists():
        raise FileExistsError(f"이미 있는 백업을 덮어쓰지 않습니다: {archive}")

    original = count_events()
    if original < 1:
        raise ValueError("기존 AI 이벤트가 1건 이상 있어야 복원 실습을 진행합니다.")

    # subprocess 출력 전체를 메모리에 담지 않고 운영 폴더의 백업 파일로 스트리밍합니다.
    try:
        with archive.open("xb") as stream:
            subprocess.run(COMPOSE + ['exec pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc', 'sh'],
                           cwd=ROOT, stdout=stream, check=True)
        with archive.open("rb") as stream:
            if stream.read(5) != b"PGDMP":
                raise ValueError("PostgreSQL custom format 백업 헤더가 없습니다.")
        with archive.open("rb") as stream:
            # --list로 백업 목차를 실제로 읽는지 검사합니다.
            checked = subprocess.run(COMPOSE + ['exec pg_restore --list', 'sh'],
                                     cwd=ROOT, stdin=stream, capture_output=True, check=True)
        if b"inference_events" not in checked.stdout:
            raise ValueError("백업 파일에 inference_events 테이블 정보가 없습니다.")
    except Exception:
        # 실패한 백업은 오인하지 않도록 .failed 이름으로 남깁니다.
        if archive.is_file():
            archive.rename(archive.with_suffix(".failed"))
        raise

    created = False
    cleanup_error = None
    try:
        docker_db('exec createdb -U "$POSTGRES_USER" "$1"', drill_db)
        created = True
        with archive.open("rb") as stream:
            subprocess.run(COMPOSE + ['exec pg_restore -U "$POSTGRES_USER" -d "$1" '
                                     '--no-owner --no-privileges', 'sh', drill_db],
                           cwd=ROOT, stdin=stream, check=True)
        restored = count_events(drill_db)
        after = count_events()
        if original != restored or original != after:
            raise AssertionError(f"건수 불일치: 원본={original}, 복원={restored}, 사후={after}")
        return {"result": "PASS", "original": original, "restored": restored,
                "after": after, "archive": str(archive), "temporary_database": drill_db}
    finally:
        if created:
            if not DRILL_NAME.fullmatch(drill_db):
                raise RuntimeError("임시 DB 이름 안전 검증에 실패했습니다.")
            try:
                docker_db('exec dropdb -U "$POSTGRES_USER" --if-exists -- "$1"', drill_db)
            except subprocess.CalledProcessError as exc:
                cleanup_error = exc
            if cleanup_error:
                raise RuntimeError(f"임시 DB 정리 실패: {drill_db} 수동 확인 필요") from cleanup_error


def main() -> None:
    parser = argparse.ArgumentParser(description="교과13 임시 DB 복원 실습")
    parser.add_argument("--backup-dir", default=str(Path.home() / "subject13-deploy/state/backups"))
    parser.add_argument("--dry-run", action="store_true", help="실제 데이터베이스에는 접속하지 않고 실행 계획만 표시")
    args = parser.parse_args()
    target = Path(args.backup_dir).expanduser().resolve()
    stamp = datetime.now(timezone.utc).strftime("%Y%m%d%H%M%S")
    if args.dry_run:
        print("DRY_RUN (PASS가 아님): 운영 DB dump → 격리 DB 복원 → COUNT 비교 → 임시 DB 삭제")
        print("백업 폴더:", target)
        return
    report = perform_drill(target, stamp)
    print("PASS: 운영 DB 레코드 보존 + 격리 DB 복원 완료")
    print("원본/복원/사후 건수:", report["original"], report["restored"], report["after"])
    print("백업 파일 (Git 제외):", report["archive"])
    print("임시 DB 삭제 완료:", report["temporary_database"])


if __name__ == "__main__":
    main()
```

**코드 이해하기**

`pg_dump -Fc`는 PostgreSQL 전용 백업 형식을 생성합니다. `PGDMP`는 그 백업 파일의 헤더입니다. `pg_restore --list`는 실제 복원하기 전에 백업 안의 테이블 정보가 읽히는지 검사합니다.

`subject13_drill_YYYYMMDDHHMMSS` 형태의 임시 데이터베이스만 생성·삭제하도록 제한하므로, `subject13_platform` 같은 실제 서비스 DB를 삭제하지 않습니다. `original`, `restored`, `after` 세 건수가 같아야 합니다. **실패 후 임시 DB를 삭제하지 못했다면 해당 이름을 기록하고 수동으로 확인**해야 합니다.

### 실습 8. 백업 실습의 실행 계획을 먼저 확인합니다

**[개발 PC]** 서버 없이 실행 방법을 살펴봅니다.

```bash
python scripts/13_db_restore_drill.py --dry-run
```

출력 예:

```text
DRY_RUN (PASS가 아님): 운영 DB dump → 격리 DB 복원 → COUNT 비교 → 임시 DB 삭제
```

이 출력은 **실제 백업이나 복원을 수행하지 않았다는 뜻**입니다. 실습 후반부에서 **[AWS EC2]**의 기존 PostgreSQL 서비스가 실행된 상태로 이 프로그램을 다시 실행합니다.

---

## PART 4. 운영 문서 양식을 만들고 Git에 검증 버전을 기록합니다

### 실습 9. Acceptance 결과 보고서 양식을 작성합니다

오늘의 점검 결과는 실제 숫자와 실행 시각을 기록합니다. 자동검사에 포함되지 않은 새 이미지 분석, Celery 파일 생성, 장비와 복원·장애 실습도 반드시 따로 남깁니다.

**파일: `docs/day13/ACCEPTANCE_REPORT_TEMPLATE.md`**

```markdown
# 교과 13 · 13일차 Acceptance Test 결과 보고서

- 검사 일시: (실제 날짜·시간)
- 프로젝트: subject13_ai_platform
- 검증 Git SHA: (EC2 git rev-parse HEAD 결과)
- 검사 환경: (개발 PC / AWS EC2 GPU / Jetson / Raspberry Pi)
- 검사자: (본인 또는 팀명)
- 보고 상태: 진행 중 / 재검증 필요 / 최종 승인

## 1. 자동 점검

- `scripts/13_acceptance.py` 결과: NOT_RUN
- 자동 점검 JSON 보관 위치: docs/day13/evidence/acceptance.json
- 1~12일차 회귀 테스트: NOT_RUN
- 13일차 신규 검사 로직 테스트: NOT_RUN

## 2. 실제 통합 점검

| 번호 | 실제로 수행할 확인 | 상태 | 확인한 결과·증빙 |
|---|---|---|---|
| E2E-01 | 교과9 OCR 실제 이미지 → OK/NG·DB | NOT_RUN | |
| E2E-02 | 교과10 Pose·Action 실시간 처리·DB | NOT_RUN | |
| E2E-03 | AWS 실제 JPG → YOLO Pose CUDA → DB | NOT_RUN | |
| E2E-04 | Redis·Celery Export SUCCESS → Bronze/Silver/Gold | NOT_RUN | |
| E2E-05 | Jetson TensorRT → 실제 카메라·Keypoint·FPS | NOT_RUN | |
| E2E-06 | Raspberry Pi ONNX → GPIO·PC 전송 | NOT_RUN | |
| E2E-07 | Prometheus Target UP·Grafana 대시보드 | NOT_RUN | |
| E2E-08 | PostgreSQL 임시 DB 복원·건수 대조 | NOT_RUN | |
| E2E-09 | Worker 일시 중단 감지·재시작·작업 성공 | NOT_RUN | |
| E2E-10 | CI/CD 커밋 SHA·Rollback 모의 검증 | NOT_RUN | |

## 3. 실제 성능 측정

- AWS GPU 추론시간(ms): 미측정
- Jetson 실제 FPS: 미측정
- Jetson 추론 p50/p95(ms): 미측정
- Raspberry Pi 추론시간(ms): 미측정
- 오류 및 복구에 걸린 시간: 미측정

## 4. 보안·데이터 보존

- .env·비밀번호·원본 영상·모델·DB 백업 Git 제외: NOT_RUN
- 기존 DB 레코드 보존: NOT_RUN
- 공개 포트·SSH 터널 점검: NOT_RUN

## 5. 발견한 문제 및 후속 조치

| 현상 | 원인/확인 명령 | 조치/담당 | 재시험 상태 |
|---|---|---|---|
| (실제 내용) | | | |

## 6. 인수 결정

- 중앙 플랫폼(EC2/API/DB/Worker) 상태: 미판정
- Edge 장치(Pi/Jetson) 상태: 미판정
- 전체 프로젝트 상태: 미판정
- 미완료·BLOCKED 항목과 이유:
- 재검증 예정 항목:

> 실제 검증하지 않은 기능을 PASS로 바꾸지 않습니다.
> AWS Pose의 UNKNOWN은 동작인식 실패가 아니라 Pose-only 구조에 따른 판정 유보입니다.
```

`NOT_RUN` 부분은 다음 실습에서 실제 결과를 확인한 뒤에만 수정합니다. **최종 작성한 결과 보고서는 운영 기록이므로 `docs/day13/evidence/ACCEPTANCE_REPORT.md`에 별도로 저장**합니다. 보고서 안에 비밀키와 개인정보를 넣지 않습니다.

### 실습 10. 운영자가 사용할 Runbook을 작성합니다

Runbook은 서비스가 중단되거나 다른 컴퓨터에서 점검할 때 **어떤 순서와 명령으로 확인할지** 기록하는 문서입니다. 기존 코드를 다시 작성하는 설명이 아니라, 실제 운영에 필요한 절차를 정리합니다.

**파일: `docs/day13/RUNBOOK.md`**

```markdown
# 교과 13 AI 플랫폼 운영 Runbook

## 1. 실행 대상

- 중앙 플랫폼: FastAPI + PostgreSQL + Redis + Celery
- GPU 서버: AWS EC2 CUDA YOLO11n-Pose
- 장비: Jetson TensorRT Pose, Raspberry Pi TinyCNN ONNX
- 모니터링: Prometheus + Grafana
- 배포: GitHub Actions + AWS SSM

## 2. 중앙 플랫폼 시작·상태 확인 (기존 EC2)

    cd ~/ai_vision/subject13_ai_platform
    git rev-parse HEAD
    bash scripts/12_dc.sh config --quiet
    bash scripts/12_dc.sh ps
    curl -fsS http://127.0.0.1:8000/health
    curl -fsS http://127.0.0.1:8000/api/v1/containers/status
    curl -fsS http://127.0.0.1:8000/api/v1/cloud/health

서비스가 중단되어 있고 정상 복구가 허용된 경우:

    bash scripts/12_dc.sh up -d --no-build

GPU 컨테이너가 시작되지 않으면 임의로 CPU 추론 모드로 바꾸지 않습니다.

## 3. 로그 확인

    bash scripts/12_dc.sh logs --tail=80 api
    bash scripts/12_dc.sh logs --tail=80 db
    bash scripts/12_dc.sh logs --tail=80 redis
    bash scripts/12_dc.sh logs --tail=80 worker
    bash scripts/12_dc.sh logs --tail=80 cloud-gpu
    bash scripts/12_dc.sh logs --tail=80 prometheus
    bash scripts/12_dc.sh logs --tail=80 grafana

로그를 외부에 전달하기 전에 토큰·비밀번호·개인정보가 없는지 확인합니다.

## 4. DB 점검·백업·복원

    bash scripts/12_dc.sh exec -T db sh -c \
      'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT COUNT(*) FROM inference_events;"'
    python3 scripts/13_db_restore_drill.py --dry-run
    python3 scripts/13_db_restore_drill.py

실제 복원은 격리 DB로만 수행합니다. 운영 DB를 직접 덮어쓰지 않습니다.
백업 위치: ~/subject13-deploy/state/backups/

## 5. 배포 버전·자동 배포

    git rev-parse HEAD
    curl -fsS http://127.0.0.1:8000/api/v1/deploy/version
    cat ~/subject13-deploy/state/last_good_sha

Git SHA 3개가 동일한지 확인합니다. 배포 실패는 GitHub Actions·SSM 기록에서 확인합니다.

## 6. 모니터링

- Prometheus: localhost:9090 (SSH 터널 권장)
- Grafana: localhost:3000 (SSH 터널 권장)
- Cloud GPU Pose: 실제 JPG/PNG 추론 후 저장 여부 확인
- Jetson 카메라: 실제 최근 프레임·실측 FPS 확인
- Raspberry Pi: 실제 GPIO LED와 PC 수신 기록 확인

## 7. 복구 시 주의

- Docker 네임드 볼륨과 모델 파일을 삭제하지 않습니다.
- `docker compose down -v`를 실행하지 않습니다.
- `.env`나 SSH 키를 GitHub에 올리지 않습니다.
- 카메라 장애로 오래된 영상을 최신으로 표시하지 않습니다.
- Rollback은 코드·컨테이너를 되돌리며 DB 기록을 자동 복원하지 않습니다.
- 실습 종료 시 AWS EC2 비용과 인스턴스 상태를 확인합니다.
```

### 실습 11. 프로젝트 인수인계 문서를 작성합니다

**파일: `docs/day13/HANDOVER.md`**

```markdown
# 교과 13 AI 서비스 플랫폼 인수인계

## 1. 인수 대상

- 프로젝트: subject13_ai_platform (1~13일차 동일 저장소)
- 소스 코드 기준: 최종 Git 커밋 SHA를 인수시 확인
- 운영 서버: 기관에서 승인한 EC2 주소와 인스턴스 ID (비공개 관리)
- 장치: Raspberry Pi, Jetson Orin Nano Super
- 데이터: PostgreSQL inference_events, CameraSource/AIModel 관계

## 2. 주요 데이터 흐름

1. 교과9 OCR 및 교과10 Action 결과 → 공통 InferenceEvent
2. FastAPI → PostgreSQL 저장·조회
3. Redis·Celery → Bronze JSONL / Silver Parquet / Gold 집계
4. Raspberry Pi ONNX → GPIO·PC 수신 기록
5. Jetson TensorRT Pose → PC API·DB
6. AWS GPU Pose → Cloud API → PostgreSQL
7. Prometheus·Grafana → 모니터링
8. GitHub Actions → AWS SSM → 배포 검증·Rollback

## 3. 아직 구현 범위가 아닌 것

- AWS YOLO Pose 출력만으로 산업안전 행동 판정을 확정하지 않습니다.
- Raspberry Pi 교육용 카드 분류기를 산업용 불량 검사 정확도로 주장하지 않습니다.
- 코드 테스트와 실제 장비 Acceptance를 같은 것으로 기록하지 않습니다.

## 4. 필수 보관 항목

- Git: 소스·설정 템플릿·운영 문서
- 기관 내부 별도 저장: 실제 .env, 인증키, DB 백업, 모델 가중치, 영상·이미지
- 인수할 때: 배포 SHA, 모델 SHA256, DB 백업 경로, 장비 연결 설정, 장애 이력

## 5. 첫 운영 점검 순서

1. git SHA와 Compose 서비스 확인
2. PostgreSQL·Redis·Celery 응답 확인
3. 실제 이미지 AI 추론과 이벤트 DB 저장 확인
4. Prometheus·Grafana 실제 수집 확인
5. 백업·복원 검증 결과 확인
6. 미해결 FAIL/BLOCKED 항목 확인

## 6. 후속 프로젝트 연결

- 교과 14: Jetson + 산업용 카메라 + 컨베이어 제조검사로 확장
- 교과 15: 교과10 Pose·Action Recognition을 현장 산업안전 서비스로 확장

## 7. 최종 인수 승인

- 검사 결과 파일: docs/day13/evidence/ACCEPTANCE_REPORT.md (운영 보관)
- 합격하지 못한 항목: 인수보고서에 상태·원인·재검증 방법 기재
- 담당자·승인자·검토 일시: 인수 시 직접 기입
```

### 실습 12. 실제 증빙 자료가 Git에 들어가지 않도록 합니다

1~12일차에 만든 `.gitignore`의 내용을 삭제하지 않고, 마지막에 다음 규칙을 추가합니다.

**파일: `.gitignore` (맨 아래에 추가)**

```gitignore
# 교과13 13일차: 실제 서비스 검사 기록과 증빙은 별도로 관리
docs/day13/evidence/
```

이 폴더에는 검사 당시의 Git SHA, 장치 상태, 백업·실측 결과가 보관될 수 있으므로 **공개 저장소에 자동 추가하지 않습니다.** `.gitignore`가 이미 Git에 추적 중인 파일을 제외시키지는 않는 점에도 주의하세요.

### 실습 13. README와 구조도에 13일차를 연결합니다

기존 `README.md`와 `docs/architecture.md`의 1~12일차 기록은 그대로 둡니다. 마지막에 아래 내용을 추가합니다.

**`README.md` 맨 아래에 추가:**

```markdown
## 13일차 — 최종 인수검사와 운영 인계

- 점검 스크립트: `scripts/13_acceptance.py`
- PostgreSQL 격리 복원: `scripts/13_db_restore_drill.py`
- 점검 테스트: `tests/test_day13.py`
- Acceptance 결과 템플릿: `docs/day13/ACCEPTANCE_REPORT_TEMPLATE.md`
- 운영 절차: `docs/day13/RUNBOOK.md`
- 인수인계: `docs/day13/HANDOVER.md`

실제 점검 결과는 `docs/day13/evidence/`에 보관하며 Git에는 포함하지 않습니다.
실제 장비가 없거나 네트워크가 연결되지 않은 항목은 `BLOCKED`로 기록합니다.
```

**`docs/architecture.md` 맨 아래에 추가:**

```markdown
## 13일차 — 최종 Acceptance 흐름

기존 FastAPI + PostgreSQL + Redis + Celery
                  ↓
실제 OCR / Action / Raspberry Pi / Jetson / AWS GPU
                  ↓
Prometheus · Grafana + Git CI/CD
                  ↓
Acceptance Test (API·추론·DB·복원·복구)
                  ↓
결과 보고서 + Runbook + 인수인계

주의: AWS YOLO Pose-only는 산업안전 Action 판단과 분리합니다.
```

### 실습 14. 13일차 검증을 CI에 연결하고 Git에 기록합니다

12일차 GitHub Actions 파일을 유지하면서 **13일차 신규 테스트를 CI 대상에 한 줄 추가**합니다. 기존 CI/CD Workflow 전체를 새로 작성할 필요는 없습니다.

**파일: `.github/workflows/day12-ci-cd.yml` (기존 `ci` 작업의 테스트 부분만 수정)**

기존 `- name: Regression tests for days 8 to 12` 부근을 찾아 테스트 파일 목록의 마지막에 `tests/test_day13.py`를 추가합니다.

```yaml
      - name: Regression tests for days 8 to 13
        run: |
          python -m pytest -q \
            tests/test_day08.py tests/test_day09.py \
            tests/test_day10_config.py tests/test_day10_jetson_route.py \
            tests/test_day10_monitoring.py tests/test_day10_telemetry.py \
            tests/test_day11_cloud.py tests/test_day12_pipeline.py \
            tests/test_day13.py
```

**위 블록은 기존 Workflow의 테스트 부분만 바꾸는 코드**입니다. `on`, `permissions`, `concurrency`, `deploy` 부분은 12일차 그대로 둡니다. CI에서 신규 테스트가 실제로 실행되지 않으면 13일차 테스트까지 검증했다고 기록할 수 없습니다.

13일차 코드를 EC2에서 실행하려면 먼저 **같은 소스 버전을 Git에 기록**해야 합니다. 다음 명령은 공개가 허용된 소스 코드에 대해서만 실행합니다.

**[개발 PC]**

```bash
cd ~/ai_vision/subject13_ai_platform
python -m pytest -q tests/test_day13.py
python scripts/12_ci_verify.py
git status --short
```

`git status`에서 `.env`, 실제 모델, 영상, 백업, `docs/day13/evidence` 등이 추가 대상이 아닌지 확인합니다. 문제가 없다면 다음처럼 **13일차에 새로 만든 파일과 기존 README·구조도 수정사항만** 추가합니다.

```bash
git add scripts/13_acceptance.py scripts/13_db_restore_drill.py \
  tests/test_day13.py docs/day13/ACCEPTANCE_REPORT_TEMPLATE.md \
  docs/day13/RUNBOOK.md docs/day13/HANDOVER.md \
  .gitignore README.md docs/architecture.md \
  .github/workflows/day12-ci-cd.yml

git diff --cached --stat
git commit -m "교과13 13일차: Acceptance Test 및 운영 인수인계"
git rev-parse HEAD
```

**승인된 원격 Git 사용이 가능한 환경에서만** 다음을 실행합니다.

```bash
git push origin main
```

12일차 설정에서 `DAY12_DEPLOY_ENABLED=true`인 경우, GitHub Actions의 **CI 통과 후 CD가 EC2 배포를 수행**합니다. GitHub의 `Actions` 화면에서 실제 커밋 SHA와 성공 상태를 확인하세요. CI가 실패하면 EC2 검사로 진행하지 않습니다.

GitHub 사용이 제한된 폐쇄망에서는 허용된 내부 Git 저장소와 기관 승인 배포 절차를 사용합니다. **GitHub Actions·SSM을 실행하지 않았다면 그 부분은 PASS로 표시하지 않습니다.**

---

## PART 5. AWS에서 실제 서비스 통합 검사를 진행합니다

### 실습 15. EC2의 실제 배포 SHA와 전체 컨테이너를 확인합니다

**[AWS EC2]** 12일차와 동일한 서버에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
export DEPLOYED_SHA="$(git rev-parse HEAD)"
echo "배포 SHA: $DEPLOYED_SHA"
cat "$HOME/subject13-deploy/state/last_good_sha"
bash scripts/12_dc.sh ps
curl -fsS http://127.0.0.1:8000/api/v1/deploy/version
```

- `git rev-parse HEAD`의 실제 SHA와 `last_good_sha`, FastAPI 응답 `commit`을 비교합니다.
- `db`, `redis`, `api`, `worker`, `prometheus`, `grafana`, `cloud-gpu`의 상태를 확인합니다.
- SHA가 다르면 `APP_COMMIT_SHA`와 배포 이력을 확인하고 `FAIL`로 기록합니다.
- `.env` 전체 값을 화면에 출력하지 않습니다.

### 실습 16. Acceptance 자동 점검을 실제 EC2에서 실행합니다

**[AWS EC2]**

```bash
cd ~/ai_vision/subject13_ai_platform
export DEPLOYED_SHA="$(git rev-parse HEAD)"
python3 scripts/13_acceptance.py \
  --expected-sha "$DEPLOYED_SHA" \
  --output docs/day13/evidence/acceptance.json
```

예상 출력의 **형식**은 다음과 같습니다. 이는 합격 예시이며 실제 서비스가 연결되지 않았다면 결과가 달라집니다.

```text
AT-01 PASS    FastAPI 기본 상태
AT-02 PASS    배포 SHA 일치
AT-03 PASS    PostgreSQL SQLAlchemy 연결
...
필수 항목 종합: PASS | 상세 기록: docs/day13/evidence/acceptance.json
```

이 프로그램은 **필수 중앙 AWS 서비스**를 점검합니다. `AT-10` Jetson과 `AT-11` Raspberry Pi는 각 장치가 실제로 연결되어 있는 경우 필요에 따라 필수로 설정합니다.

**Jetson·Pi까지 같은 API에서 검사할 수 있는 환경일 때만:**

```bash
python3 scripts/13_acceptance.py \
  --expected-sha "$DEPLOYED_SHA" \
  --require-jetson --require-pi \
  --output docs/day13/evidence/acceptance_all_devices.json
```

AWS에서 학교 내부망의 Jetson·Pi에 접근할 수 없다면 기기 검사에 `BLOCKED`를 기록하고, 장비가 있는 강의실 PC에서 각각 별도로 점검하세요. **옵션을 빼고 중앙 서버 검사만 통과한 것을 전체 하드웨어 통합 합격으로 해석하지 않습니다.**

### 실습 17. 실제 Pose 이미지를 AWS GPU에 넣고 DB 저장까지 확인합니다

자동 상태검사는 이미 기록된 결과를 조회하므로, **새 이미지의 실제 추론**을 직접 확인해야 합니다. 11일차에 사용한 **저작권·보안 문제가 없는 사람 사진(JPG 또는 PNG)**을 사용합니다. 실제 사람이 한 명 이상 나타나는 사진이 판정에 적합합니다.

**[개발 PC]** EC2에 SSH 터널을 연결합니다. 기존 PC의 포트 8000이 이미 사용 중이라면 해당 서비스를 안전하게 중지하거나, 이전 실습에서 허용된 터널 포트 설정을 조정합니다.

```bash
export EC2_PUBLIC_IP="본인_EC2_공인_IP"
export SSH_KEY="$HOME/.ssh/subject13-day11.pem"
ssh -i "$SSH_KEY" -N \
  -L 8000:127.0.0.1:8000 \
  -L 3000:127.0.0.1:3000 \
  -L 9090:127.0.0.1:9090 \
  ubuntu@"$EC2_PUBLIC_IP"
```

SSH 터널 터미널은 그대로 둡니다. 브라우저에서 **`http://127.0.0.1:8000/cloud`**를 열고 실제 JPG/PNG 파일을 업로드하여 분석합니다.

같은 실습을 `curl`로 확인하려면 **다른 [개발 PC] 터미널**에서 다음처럼 실행합니다(아래는 실제 JPG 파일을 사용하는 경우).

```bash
export POSE_TEST_IMAGE="$HOME/ai_vision/test_images/pose_person.jpg"
ls -lh "$POSE_TEST_IMAGE"
curl -fsS -X POST http://127.0.0.1:8000/api/v1/cloud/analyze \
  -F "image=@${POSE_TEST_IMAGE};type=image/jpeg" | python -m json.tool
```

PNG 파일을 사용하는 경우 MIME 타입을 `image/png`로 바꾸세요. **`POSE_TEST_IMAGE` 경로는 학생이 실제로 보유한 허용된 이미지 경로**로 수정해야 하며, 파일이 없다면 성공 응답을 만들어 내지 않습니다.

정상적인 실제 검증에서는 다음을 확인합니다.

1. HTTP 요청이 성공하고 `saved`와 `event`가 반환되는가?
2. `event.event_id`는 `cloud:`로 시작하는가?
3. `event.source_mode`는 `live_inference`인가?
4. `event.details.runtime`이 `pytorch_cuda`이고 `model_sha256`이 기록되는가?
5. 사람 사진이라면 `num_persons`와 `persons[].keypoints`에서 검출 결과를 확인할 수 있는가? 검출 실패라면 정확한 입력·신뢰도를 기록하는가?
6. **`decision`이 `UNKNOWN`인 것은 정상일 수 있습니다.** AWS는 Pose-only이며 위험행동 분류가 아닙니다.

**[AWS EC2]** 새로 만든 결과가 DB에 저장되었는지도 확인합니다.

```bash
curl -fsS 'http://127.0.0.1:8000/api/v1/cloud/events?limit=20' \
  | python3 -m json.tool

bash scripts/12_dc.sh exec -T db sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c \
  "SELECT event_id, task, decision, source_mode FROM inference_events WHERE event_id LIKE '\''cloud:%'\'' ORDER BY id DESC LIMIT 5;"'
```

**새로 받은 `event_id`와 DB 조회에 나오는 `event_id`가 일치해야 E2E-03에 PASS**를 기록합니다. 단순히 `/cloud/health`가 `connected=true`인 것만으로 이미지 추론 성공 처리하지 않습니다.

### 실습 18. Celery Export로 Bronze·Silver·Gold가 생성되는지 확인합니다

**[AWS EC2]** 다음 명령으로 기존 비동기 API를 확인합니다.

```bash
curl -fsS http://127.0.0.1:8000/api/v1/pipeline/health
```

이제 실제 Export 작업을 등록합니다.

```bash
curl -fsS -X POST http://127.0.0.1:8000/api/v1/pipeline/exports \
  -H 'Content-Type: application/json' \
  -d '{"task":null,"limit":30}' | python3 -m json.tool
```

서버는 성공적으로 접수하면 작업 ID를 돌려줍니다. 예를 들어 `job_id`가 `123e4567-e89b-12d3-a456-426614174000`이었다면 다음처럼 조회합니다. **실제 응답으로 받은 ID를 넣으세요.**

```bash
export JOB_ID="실제_응답으로_받은_UUID"
curl -fsS "http://127.0.0.1:8000/api/v1/pipeline/jobs/$JOB_ID" \
  | python3 -m json.tool
```

상태는 `PENDING`에서 `STARTED`, `SUCCESS` 등으로 바뀔 수 있습니다. **HTTP 202 또는 `PENDING`만으로 성공한 것이 아닙니다.** 작업 결과가 `SUCCESS`이고, `data/day03/exports/<작업ID>/`에 다음 파일이 실제로 있어야 합니다.

```text
data/day03/exports/<실제 작업 ID>/
├── bronze_events.jsonl
├── silver_events.parquet
├── gold_summary.json
└── manifest.json
```

**[AWS EC2]** 결과 파일을 확인합니다.

```bash
ls -lh "data/day03/exports/$JOB_ID/"
python3 - "$JOB_ID" <<'PY'
import json, sys
from pathlib import Path
root = Path('data/day03/exports') / sys.argv[1]
manifest = json.loads((root / 'manifest.json').read_text(encoding='utf-8'))
gold = json.loads((root / 'gold_summary.json').read_text(encoding='utf-8'))
for name in ('bronze_events.jsonl', 'silver_events.parquet',
             'gold_summary.json', 'manifest.json'):
    assert (root / name).is_file() and (root / name).stat().st_size > 0, name
assert manifest['row_count'] == gold['total']
print('PASS: 실제 Bronze·Silver·Gold·Manifest 생성과 건수 일치')
print('Export 건수:', manifest['row_count'])
PY
```

Parquet의 실제 데이터 형식과 행 수 검증이 필요하면 **3일차에 작성한 Parquet 검증 코드**도 다시 실행합니다. 13일차는 새 변환 알고리즘을 만들지 않습니다.

### 실습 19. Prometheus와 Grafana의 수집 상태를 확인합니다

**[AWS EC2]**

```bash
bash scripts/12_dc.sh ps
bash scripts/12_dc.sh logs --tail=30 prometheus
bash scripts/12_dc.sh logs --tail=30 grafana
```

**[개발 PC]** 위에서 만든 SSH 터널을 유지하고 다음 URL을 엽니다.

```text
http://127.0.0.1:9090/targets
http://127.0.0.1:3000/
```

Prometheus `Targets`에서 `platform_api`, `cloud_gpu`, `prometheus`가 **UP**인지 확인합니다. PromQL에서 다음을 실행합니다.

```promql
up{job="platform_api"}
```

```promql
up{job="cloud_gpu"}
```

정상일 때 각 결과는 `1`입니다. Grafana에서 10~11일차에 만든 대시보드가 유지되는지 확인하고, 오늘의 실제 Cloud GPU 이미지 추론으로 증가한 요청 수·실측 추론시간을 기록합니다. 값이 비어 있다면 임의 수치를 적지 말고 `미측정` 또는 `FAIL/BLOCKED`로 기록합니다.

---

## PART 6. 복원·장애·장치 검증을 완료합니다

### 실습 20. PostgreSQL의 실제 백업·임시 DB 복원을 검사합니다

**[AWS EC2]** 신뢰할 수 있는 기존 DB와 충분한 디스크 공간이 있는지 확인합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
df -h "$HOME"
bash scripts/12_dc.sh exec -T db sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT COUNT(*) FROM inference_events;"'
python3 scripts/13_db_restore_drill.py --dry-run
```

준비가 되었으면 다음을 실행합니다. 기존 DB에 실제로 연결하므로 **DB 백업과 임시 DB 생성 권한이 있는 승인된 실습 환경에서만** 진행합니다.

```bash
python3 scripts/13_db_restore_drill.py
```

성공한 경우 출력 형식은 다음과 같습니다.

```text
PASS: 운영 DB 레코드 보존 + 격리 DB 복원 완료
원본/복원/사후 건수: (실제 원본 수) (실제 복원 수) (실제 사후 수)
백업 파일 (Git 제외): /home/ubuntu/subject13-deploy/state/backups/day13_acceptance_....dump
임시 DB 삭제 완료: subject13_drill_....
```

세 건수가 실제로 같을 때만 `E2E-08`에 PASS를 기록합니다. `PGDMP` 헤더가 맞아도 데이터 복원이 실패할 수 있으므로, **백업 파일 존재만으로 복원 성공이라고 판단하지 않습니다.**

**주의:** 복원 중 디스크 부족이나 권한 오류가 발생하면 `FAIL`을 기록하고, 기존 운영 DB를 손대지 않은 상태에서 원인을 해결하세요. 남아 있는 `subject13_drill_...` 임시 DB가 있다면 이름과 상태를 확인한 후 정리합니다.

### 실습 21. Celery Worker 장애와 복구를 확인합니다

**이 실습은 다른 팀이 사용하지 않는 승인된 실습 환경에서만 수행합니다.** 정상 사용 중인 운영 서비스에서 예고 없이 Worker를 중단하지 않습니다. DB나 GPU 컨테이너를 중단할 필요는 없습니다.

**[AWS EC2]** 중단 전 상태를 확인합니다.

```bash
curl -fsS http://127.0.0.1:8000/api/v1/containers/status
bash scripts/12_dc.sh ps
```

점검 시간에 Worker만 중단합니다.

```bash
bash scripts/12_dc.sh stop worker
curl -fsS http://127.0.0.1:8000/api/v1/containers/status
```

`celery_worker_responding`이 `false`가 되거나 API 상태가 `degraded`로 바뀌는지 확인합니다. Worker를 정지했다고 이미 DB에 저장된 기록이 사라져서는 안 됩니다.

다시 시작합니다.

```bash
bash scripts/12_dc.sh start worker
bash scripts/12_dc.sh logs --tail=50 worker
curl -fsS http://127.0.0.1:8000/api/v1/containers/status
```

Worker가 다시 `true`로 응답할 때까지 상태를 점검합니다. **실제 복구 완료는 새로운 Export를 한 번 더 등록해 `SUCCESS`와 파일 생성까지 확인한 후** 기록합니다. 재시작 명령만 성공했다고 복구 완료로 처리하지 않습니다.

### 실습 22. Jetson·Raspberry Pi와 교과 9·10 기능을 다시 확인합니다

PC에서 AWS로 연결한 SSH 터널을 쓰고 있다면 8000 포트의 대상을 확인합니다. 교과 9·10의 **PC 플랫폼**을 검사할 때는 AWS 터널을 종료하고 PC의 기존 FastAPI를 실행합니다.

**[개발 PC]** 기존 Docker/가상환경에 맞게 서비스를 실행한 후:

```bash
curl -fsS http://127.0.0.1:8000/api/v1/live/health
curl -fsS 'http://127.0.0.1:8000/api/v1/live/inspection/samples'
curl -fsS 'http://127.0.0.1:8000/api/v1/live/events' | python3 -m json.tool
```

브라우저의 `/live`에서 실제 OCR 이미지 검사와 교과 10 IP Webcam Action Recognition을 각각 진행합니다. 결과가 기록되면 `POST /api/v1/live/inspection/run`, `POST /api/v1/live/action/save`와 PostgreSQL 기록을 확인합니다. **실제 카메라·OCR 이미지·RF 모델 파일이 없다면 `BLOCKED`**로 기록합니다.

**[Jetson과 연결된 PC 플랫폼]** Jetson의 기존 TensorRT 서비스가 실행 중일 때:

```bash
curl -fsS http://127.0.0.1:8000/api/v1/jetson/health
curl -fsS http://127.0.0.1:8000/api/v1/jetson/stream/latest | python3 -m json.tool
```

`/jetson-stream` 화면에서 실제 최근 프레임, Bounding Box, Keypoint, FPS를 확인하고, 이전 9일차의 저장 버튼으로 PostgreSQL 기록을 확인합니다. **TensorRT Runtime이 실제 사용 중인지** 확인하세요. 예전 파일을 화면에 띄우는 것만으로 실시간 합격이 아닙니다.

**[Raspberry Pi와 연결된 PC 플랫폼]** 실제 Pi에서 TinyCNN 추론·LED GPIO를 실행하고:

```bash
curl -fsS http://127.0.0.1:8000/api/v1/edge-device/health
curl -fsS 'http://127.0.0.1:8000/api/v1/edge-device/events?limit=10' \
  | python3 -m json.tool
```

- `normal` 카드에서 파란 LED, `warning_red` 카드에서 빨간 LED, UNKNOWN에서 모두 OFF인지 확인합니다.
- Pi의 출력 결과가 PC PostgreSQL에 `pi:` 이벤트로 들어오는지 확인합니다.
- 장치가 없으면 이미지·LED·성능을 임의로 합격 처리하지 않습니다.

Jetson, Pi, 교과 9·10은 EC2 CUDA 서비스와 실행환경이 다르므로, **각 기기별로 실제 결과와 상태를 따로 기록**합니다.

### 실습 23. CI/CD와 Rollback 결과를 확인합니다

**[개발 PC]** 12일차에 작성한 실제 Git 저장소 Rollback 테스트를 실행합니다.

```bash
DATABASE_URL='sqlite+pysqlite:///:memory:' \
  python -m pytest -q tests/test_day12_remote_rollback.py -v
```

정상이라면 이전에 확인한 `test_real_git_commit_deploy_success`, `test_real_git_rollback_after_new_build_failure` 등이 통과합니다. 이는 **임시 Git 저장소와 테스트용 Compose 동작의 검증**이며, 실제 EC2 GPU 서비스를 고장 내어 복구한 것은 아닙니다.

**[AWS EC2]** 실제 배포 이력을 확인합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git rev-parse HEAD
cat "$HOME/subject13-deploy/state/last_good_sha"
ls -lh "$HOME/subject13-deploy/state/backups/"
curl -fsS http://127.0.0.1:8000/api/v1/deploy/version
```

실제 EC2에서 Rollback을 실행하지 않았다면 인수보고서에는 **`Rollback 모의 검증 PASS / 운영 EC2 Rollback NOT_RUN`**으로 분리해 적습니다. 임의로 운영 AWS 서비스를 고장 내는 실습은 진행하지 않습니다.

### 실습 24. Acceptance 결과 보고서를 실제 기록으로 완성합니다

**[개발 PC 또는 승인된 내부 문서 저장 위치]** 9번 실습의 빈 템플릿을 작업용 보고서로 복사합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
cp docs/day13/ACCEPTANCE_REPORT_TEMPLATE.md \
   docs/day13/evidence/ACCEPTANCE_REPORT.md
```

`docs/day13/evidence/ACCEPTANCE_REPORT.md`를 열고 **실제로 확인한 값**을 기록합니다. AWS EC2에서 생성한 자동 검사 JSON은 기관에서 허용한 방식으로 이 증빙 폴더에 보관합니다. 원본 DB 덤프나 인증키를 공개 저장소로 복사하지 않습니다.

특히 다음 사항을 구분합니다.

- 중앙 AWS 플랫폼만 점검한 자동 `overall=PASS`는 **전체 장치 합격이 아닙니다.**
- 교과 9·10·Raspberry Pi·Jetson까지 확인하지 못한 경우 전체 인수결과는 `BLOCKED` 또는 `조건부 인계`입니다.
- 출력한 FPS·추론시간·복원 건수는 **실제로 측정한 값만** 기입합니다.
- `FAIL` 항목은 원인, 조치, 재시험 날짜 또는 실행 명령까지 적습니다.

### 실습 25. 오류가 발생하면 다음 순서로 복구합니다

| 실제 증상 | 먼저 확인할 항목 | 복구 확인 기준 |
|---|---|---|
| `AT-02 FAIL` | Git HEAD·`last_good_sha`·`APP_COMMIT_SHA` | 배포 SHA 3개 일치 |
| `AT-03/04 FAIL` | PostgreSQL 연결·`inference_events` | DB SQL 조회·기존 이벤트 보존 |
| `AT-05 FAIL` | Redis PING·Celery Worker Ping·Compose | 응답 회복 후 Export SUCCESS |
| `AT-07 FAIL` | AWS GPU CUDA·모델 경로·컨테이너 로그 | `pytorch_cuda`, 실제 이미지 추론 |
| `AT-08 FAIL` | Cloud Pose 결과 DB 저장 여부 | 실제 새 `cloud:` 이벤트 생성·조회 |
| `AT-09 FAIL` | `/metrics`·Prometheus API 상태 | 메트릭 실제 응답·Target UP |
| Jetson FAIL/BLOCKED | JetPack·TensorRT 엔진·카메라·네트워크 | 새 프레임·Keypoint·FPS 확인 |
| Raspberry Pi FAIL/BLOCKED | ONNX Runtime·카메라·GPIO·PC 수신 | LED·`pi:` DB 이벤트 확인 |
| DB 복원 FAIL | `pg_dump`/`pg_restore`·권한·여유 공간 | **임시 DB** 복원과 건수 대조 |
| Worker 복구 FAIL | Worker 로그·Redis 연결·작업 ID | 새 Export SUCCESS·결과 파일 확인 |
| GitHub CI 실패 | 테스트·추적 제외·Compose 문법 | 수정 후 CI 재실행 PASS |
| AWS Rollback 실패 | SSM·배포 상태·기존 커밋·GPU | 이전 서비스 정상 복구 후 재검사 |

**오류 해결 원칙:** 운영 DB를 초기화하거나 모델을 임의로 바꾸지 않습니다. 문제를 재현한 실제 로그와 판정 기준을 기록하고, **해당 기능을 다시 실행하여 조건을 만족했을 때** 상태를 바꿉니다.

### 실습 26. 인수인계와 Git 상태를 최종 확인합니다

**[개발 PC]**

```bash
cd ~/ai_vision/subject13_ai_platform
git status --short
git log -1 --oneline
git check-ignore docs/day13/evidence/ACCEPTANCE_REPORT.md
python -m pytest -q tests/test_day13.py
```

인수 문서(`docs/day13/RUNBOOK.md`, `docs/day13/HANDOVER.md`)가 최종 Git 기록에 포함되어 있는지 확인합니다. `git check-ignore`는 실행 증빙 폴더가 **Git 제외 대상**인지 확인하는 명령입니다.

실제 인수 결과 보고서는 허가된 내부 저장소에 보관합니다. **최종 검증 후 문서 수정만으로 다시 `main`에 Push하면 12일차 CI/CD가 다시 실행되어 배포 SHA가 달라질 수 있습니다.** 따라서 테스트가 끝난 뒤의 실측 결과는 먼저 `evidence/`에 보관하고, 새 배포가 필요한 소스 변경은 별도 검증·승인을 거칩니다.

---

# 6. 오늘 작성한 내용의 최종 정리 및 GitHub 업로드

## 6.1. 교과 13의 최종 결과를 이해합니다

1일차에는 교과 9와 교과 10의 서로 다른 AI 결과를 공통 JSON으로 정리했습니다. 2일차에는 PostgreSQL을 연결했고, 3일차에는 Redis·Celery와 Bronze·Silver·Gold 데이터 처리를 추가했습니다. 4일차에는 실제 AI 추론을 연결했습니다. 5~6일차에는 TinyCNN ONNX와 Raspberry Pi 장치를, 7일차에는 Docker Compose를 구성했습니다. 8~9일차에는 Jetson GPU·TensorRT, 10일차에는 Prometheus·Grafana, 11일차에는 AWS EC2 GPU, 12일차에는 Git CI/CD·Rollback을 만들었습니다.

13일차에는 새로운 모델 대신 **완성된 기능을 실제로 검사하고 인수할 수 있는 상태인지 판단하는 작업**을 수행했습니다.

```text
1~4일차:   AI 결과 통합 → FastAPI → DB → 실제 추론
5~6일차:   TinyCNN ONNX → Raspberry Pi CPU/GPIO
7~9일차:   Docker Compose → Jetson TensorRT → 카메라
10일차:    Prometheus → Grafana
11~12일차: AWS CUDA GPU → Git CI/CD → Rollback
13일차:    최종 Acceptance → 복원·복구 확인 → 운영 인계
```

**전체 프로젝트의 완료 여부는 보고서에 기록된 실제 PASS·FAIL·BLOCKED 결과로 결정합니다.** 학습용 자동 테스트가 통과했다는 이유만으로 전체 장치·클라우드 서비스를 검증했다고 주장하지 않습니다.

## 6.2. 최종 자가 체크리스트

- [ ] 12일차 Git 저장소와 동일한 `subject13_ai_platform`에서 13일차를 진행했습니다.
- [ ] `scripts/13_acceptance.py`와 `scripts/13_db_restore_drill.py`를 직접 작성했습니다.
- [ ] `tests/test_day13.py`의 신규 테스트 19개를 실행해 결과를 확인했습니다.
- [ ] GitHub Actions CI/CD 및 실제 EC2 배포 SHA를 확인했습니다.
- [ ] AWS GPU에 실제 이미지를 넣고 새 `cloud:` 이벤트가 DB에 저장되는지 확인했습니다.
- [ ] Celery Export가 `SUCCESS`이고 Bronze·Silver·Gold·Manifest 파일을 검증했습니다.
- [ ] Prometheus·Grafana가 실제 메트릭을 수집하는지 확인했습니다.
- [ ] PostgreSQL 백업을 임시 DB에 복원하고 원본 건수와 일치함을 확인했습니다.
- [ ] Worker 장애·재시작·새 작업 성공 여부를 확인했습니다.
- [ ] 교과 9 OCR·교과 10 Action·Jetson·Raspberry Pi의 실제 검증 결과를 각각 기록했습니다.
- [ ] 미실행·장비 미제공 항목을 PASS가 아닌 `NOT_RUN`/`BLOCKED`로 기록했습니다.
- [ ] `ACCEPTANCE_REPORT_TEMPLATE.md`, `RUNBOOK.md`, `HANDOVER.md`를 완성했습니다.
- [ ] `.env`, 원본 데이터, 모델, DB 백업, 실측 증빙을 공개 Git에서 제외했습니다.

## 6.3. 최종 GitHub 기록

13일차의 코드는 **실습 14에서 다음 메시지로 커밋하고, 12일차의 CI/CD를 통과한 버전**을 검증했습니다.

```bash
git log -1 --oneline
git status
```

커밋 메시지:

```text
교과13 13일차: Acceptance Test 및 운영 인수인계
```

이후 실측 결과는 Git에서 제외한 `docs/day13/evidence/`에 기록합니다. 기관에서 승인된 별도의 운영 기록 저장소에 공유할 수 있지만, 검증이 끝난 뒤 무심코 새 코드를 Push하여 AWS 운영 버전을 바꾸지 않도록 합니다.

## 6.4. 교과 14·15와 연결합니다

**교과 14 — 제조 AI 프로젝트:** Jetson·산업용 카메라·컨베이어를 연결하고, 실제 제조 검사 결과와 운영 지표를 지금 만든 플랫폼에 기록하도록 확장할 수 있습니다. 다만 이번 13일차의 교육용 Pose·TinyCNN 모델을 **제조 불량 검출 모델이라고 주장하지 않습니다.** 제조용 모델과 판정 규칙은 프로젝트에서 별도로 검증합니다.

**교과 15 — 산업안전 동작인식 프로젝트:** 교과 10에서 만든 38-D Feature·Sliding Window·Random Forest 또는 새롭게 검증된 Action 모델을 기반으로 실제 동작인식과 산업안전 판단을 확장합니다. 교과 13에서 완성한 DB·배포·모니터링·CI/CD·운영 인수 구조를 재사용할 수 있습니다.

## 최종 확인

**하나의 AI 플랫폼을 13일 동안 확장했습니다. 마지막으로 필요한 일은 기능을 '만들었다'고 적는 것이 아니라, 실행 결과를 근거로 '어디까지 검증했고 무엇이 남았는지' 인수할 수 있게 정리하는 것입니다.**

---

## 공식 참고 문서

- [FastAPI — Testing](https://fastapi.tiangolo.com/tutorial/testing/)
- [PostgreSQL — pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html)
- [PostgreSQL — pg_restore](https://www.postgresql.org/docs/current/app-pgrestore.html)
- [Docker Compose — CLI](https://docs.docker.com/compose/reference/)
- [Celery — Monitoring and Management](https://docs.celeryq.dev/en/stable/userguide/monitoring.html)
- [Prometheus — Querying](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Grafana — Dashboards](https://grafana.com/docs/grafana/latest/dashboards/)
- [AWS Systems Manager — Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/run-command.html)
- [GitHub Actions — Understanding GitHub Actions](https://docs.github.com/en/actions/get-started/understand-github-actions)

