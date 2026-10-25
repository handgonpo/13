## 1. 오늘의 목표

1일차에는 교과 9의 OCR 검사 결과와 교과 10의 Action Recognition 결과를 공통 `InferenceEvent`로 변환하고 웹에서 조회했습니다. 2일차에는 그 결과를 `camera_sources`, `ai_models`, `inference_events` 테이블로 나누어 PostgreSQL에 저장했습니다.

오늘은 **이미 저장된 AI 결과를 분석하기 좋은 데이터 파일로 정리하고, 파일 내보내기 작업을 웹 요청과 분리**합니다. 현재 만들어 둔 PostgreSQL과 FastAPI를 그대로 사용하므로 새 프로젝트를 만들 필요가 없습니다.

오늘 수업을 마치면 다음 작업을 직접 할 수 있습니다.

- 1·2일차의 AI 결과와 PostgreSQL 테이블을 그대로 다시 사용할 수 있습니다.
- 이미지·영상·Keypoint 같은 **원본 데이터**와 파일명·예측 결과 같은 **메타데이터**를 구분할 수 있습니다.
- 같은 AI 이벤트를 **JSONL(Bronze) → Parquet(Silver) → 요약 JSON(Gold)**으로 저장할 수 있습니다.
- OCR 결과에 없는 `confidence`를 만들지 않고 `NULL/null`로 유지할 수 있습니다.
- `UNKNOWN`을 위험 판정으로 바꾸지 않고 그대로 집계할 수 있습니다.
- Redis를 실행하고 `redis-cli ping`으로 연결 상태를 확인할 수 있습니다.
- Celery Worker를 FastAPI와 별도 터미널에서 실행할 수 있습니다.
- 웹에서 내보내기 작업을 요청하고 **작업 ID와 상태**를 조회할 수 있습니다.
- 기존 PostgreSQL의 기록을 삭제·수정하지 않고 JSONL·Parquet 결과를 생성할 수 있습니다.
- 자동 테스트와 최종 데이터 확인 후 README·Git에 오늘의 작업을 정리할 수 있습니다.

**오늘의 완성 기준:** PostgreSQL에 저장된 기존 AI 결과가 JSONL·Parquet·요약 JSON 파일로 만들어지고, FastAPI의 작업 등록 요청이 즉시 작업 ID를 반환하며, 별도의 Celery Worker가 파일 저장을 완료해야 합니다. 학생마다 원본 기록 개수는 다를 수 있습니다.

---

## 2. 오늘 수업의 전체 흐름

![[Pasted image 20261018125903.png]]

```text
1일차: 교과 9 OCR · 교과 10 Action 결과를 공통 JSON으로 변환
                               ↓
2일차: PostgreSQL inference_events에 결과 저장
                               ↓
                   3일차 프로젝트 다시 열기
                               ↓
                 PostgreSQL 기록·Redis 상태 확인
                               ↓
             이미지·영상·Keypoint 메타데이터 구분
                               ↓
             Bronze / Silver / Gold 데이터 구조 설계
                               ↓
             기존 DB 이벤트 → Bronze JSONL 저장
                               ↓
                     Silver Parquet 저장
                               ↓
                  Gold 판정별 집계 JSON 저장
                               ↓
                   동기 실행으로 파일 검증
                               ↓
                Redis Broker + Celery Worker 실행
                               ↓
            FastAPI에서 비동기 Export 요청 (HTTP 202)
                               ↓
              작업 ID 반환 → Worker 파일 저장
                               ↓
             작업 상태 조회 → SUCCESS · 산출물 확인
                               ↓
                   웹 화면·자동 테스트
                               ↓
                    README · Git Commit
                               ↓
4일차: 실제 카메라/AI 추론 결과도 동일한 저장 구조에 연결
```

**오늘은 새 이미지나 영상을 촬영하지 않습니다.** 교과 9·10에서 이미 생성해 2일차 PostgreSQL에 보관한 결과가 입력 데이터입니다. 4일차에 실제 AI 추론을 연결하기 전, 결과 저장과 비동기 처리 구조부터 완성합니다.

---

## 3. 실습 전에 알아야 할 핵심

### 3.1. 전날 만든 프로젝트에서 무엇을 사용하나요?

```text
subject13_ai_platform/
├── app/
│   ├── schemas.py          ← 1일차 공통 InferenceEvent
│   ├── records.py          ← 1일차 교과 9·10 결과 변환
│   ├── main.py             ← 1·2일차 FastAPI
│   ├── db.py               ← 2일차 DB Session
│   ├── models.py           ← 2일차 ORM 테이블 3개
│   ├── repository.py       ← 2일차 DB 저장·공통 JSON 변환
│   ├── db_api.py           ← 2일차 PostgreSQL API
│   └── db_schemas.py       ← 2일차 수동 CRUD 요청 형식
├── scripts/
│   ├── 01_import_results.py
│   ├── 02_create_tables.py
│   └── 03_import_records_to_db.py
├── web/
│   ├── index.html          ← 1일차 저장 JSON 조회
│   └── db.html             ← 2일차 PostgreSQL 조회
├── handoff/records/
│   ├── inspection_history.json
│   └── action_jobs.json
├── .env                    ← 2일차 PostgreSQL 접속정보
├── .env.example
└── requirements.txt
```

이 파일들을 다시 만들지 않습니다. 특히 `app.repository.row_to_event()`는 2일차의 DB 행을 1일차 공통 이벤트 형식으로 되돌리는 함수이므로 오늘 그대로 재사용합니다.

**2일차까지 정상적으로 따라왔다면 오늘 원본 AI 모델·OCR 이미지·MP4 파일이 없어도 됩니다.** PostgreSQL에 저장된 `recorded_result` 데이터가 있으면 됩니다. 결석 또는 미완성으로 DB 기록이 없다면 교과 9의 `service_history.json`, 교과 10의 작업별 `result.json`을 전달받아 1일차 데이터 인수와 2일차 DB 이관을 먼저 완료해야 합니다. 대체 JSON을 원본 파일명과 폴더 구조대로 배치하고, 다음 두 명령으로 DB를 복구할 수 있습니다.

```bash
python -m scripts.02_create_tables
python -m scripts.03_import_records_to_db
```

단, 이 두 명령을 실행하기 전에 2일차의 PostgreSQL과 `.env` 설정이 준비되어 있어야 합니다. 다른 학생의 `.env` 비밀번호를 복사하지 않습니다.

### 3.2. AI Native 데이터에서 원본과 메타데이터를 구분합니다

AI 서비스를 운영하면 하나의 결과에도 다음처럼 서로 다른 종류의 데이터가 연결됩니다.

```text
교과 9
제품 이미지 파일 → OCR 검사 → SN 문자열 · OK/NG · 검사 이유

교과 10
MP4 영상 → YOLO Pose → Keypoint → 38-D Feature → Action 결과
```

오늘 PostgreSQL에 들어 있는 것은 **이미지 픽셀과 영상 Frame 자체가 아니라, 검사·인식 결과에 대한 구조화된 기록**입니다. 파일명 `input_ref`만 있다고 해서 원본 이미지가 현재 PC에도 존재한다는 뜻은 아닙니다.

| 데이터 종류 | 예시 | 3일차에서 다루는 방법 |
|---|---|---|
| 이미지·영상 원본 | `.png`, `.jpg`, `.mp4` | 파일 내용은 복사하지 않고 이름과 종류만 기록 |
| Pose·Keypoint 원본 | Keypoint CSV/JSON | 오늘은 새로 생성하지 않음. 이후 파일 경로 연결 가능 |
| AI 추론 메타데이터 | `event_id`, `task`, `decision` | PostgreSQL에서 읽어 데이터 파일로 내보냄 |
| 분석용 데이터 | `confidence`, 판정, 시간, 종류 | Parquet 컬럼으로 정리 |
| 집계 데이터 | 검사 건수, OK/NG/UNKNOWN 건수 | Gold 요약 JSON으로 생성 |

위 표처럼 **파일의 실제 존재 여부와 파일을 가리키는 메타데이터는 다릅니다.** 오늘 생성하는 `asset_file_verified: false`는 파일을 확인하지 않았다는 표시이지 파일이 없다는 확정 판정이 아닙니다.

### 3.3. JSON과 JSONL은 어떻게 다른가요?

1일차에는 여러 이벤트를 하나의 JSON 배열로 다루었습니다.

```json
[
  {"event_id": "inspection:1", "decision": "OK"},
  {"event_id": "inspection:2", "decision": "NG"}
]
```

오늘 사용할 **JSONL(JSON Lines)**은 한 줄마다 완성된 JSON 객체를 저장합니다.

```jsonl
{"event_id":"inspection:1","decision":"OK"}
{"event_id":"inspection:2","decision":"NG"}
```

큰 기록을 차례대로 처리하거나 한 줄씩 읽기에 편리한 형식입니다. 오늘 실제 Bronze JSONL에는 위처럼 두 필드만 저장하는 것이 아니라, **1일차 공통 이벤트 전체와 입력 파일 메타데이터**를 함께 담습니다.

### 3.4. Parquet은 왜 사용하나요?

JSONL은 개별 이벤트 구조를 이해하고 기록을 남기기 좋습니다. 반면 예를 들어 모든 기록에서 `task`, `decision`, `confidence`만 반복해서 읽고 분석할 때는 컬럼 기반 형식인 **Parquet**이 편리합니다.

```text
행 단위 JSONL                   열 단위 Parquet

이벤트 A 전체                     task 열
이벤트 B 전체           →         decision 열
이벤트 C 전체                     confidence 열
```

Parquet은 바이너리 파일이므로 텍스트 편집기로 열어서 읽지 않습니다. 오늘은 `pyarrow`로 파일을 저장하고 다시 읽어 행 수와 열 구조를 확인합니다.

### 3.5. Bronze·Silver·Gold는 어떤 단계인가요?

오늘은 다음과 같은 **간단한 교육용 3단계 데이터 구조**를 사용합니다. 모든 데이터를 처리하는 거대한 데이터레이크를 구축하는 것은 아닙니다.

| 단계 | 실제 파일 | 역할 |
|---|---|---|
| **Bronze** | `bronze_events.jsonl` | 1일차 공통 이벤트와 출처 정보를 보존한 DB 스냅샷 |
| **Silver** | `silver_events.parquet` | 분석하기 쉽도록 자료형과 열 이름을 정리한 데이터 |
| **Gold** | `gold_summary.json` | 교과별·판정별 건수를 정리한 요약 데이터 |

`Bronze`는 센서에서 막 들어온 원본 영상이 아닙니다. **2일차 PostgreSQL에 이미 저장된 결과를 내보낸 스냅샷**입니다. `Gold`의 OK 비율이나 건수도 오늘 저장된 교육용 기록에 대한 집계일 뿐 현장 성능이나 모델 정확도를 증명하지 않습니다.

### 3.6. Redis와 Celery는 각각 무슨 일을 하나요?

지금까지는 FastAPI에서 요청을 받으면 같은 프로세스가 필요한 작업을 직접 처리했습니다. 하지만 Parquet 저장이나 다량의 결과 후처리는 시간이 걸릴 수 있습니다.

```text
[브라우저]
   │  POST /api/v1/pipeline/exports
   ▼
[FastAPI] ── 작업 메시지 ──▶ [Redis Broker]
   │                              │
   │ 작업 ID 바로 반환            ▼
   │                        [Celery Worker]
   │                              │
   ▼                              ├→ PostgreSQL 결과 조회
[브라우저]                        ├→ Bronze JSONL
   │                              ├→ Silver Parquet
   │                              └→ Gold 요약 JSON
   │                                    │
   └── 상태 조회 ◀── [Redis Result Backend] ◀── 작업 완료
```

- **Redis**: Celery가 처리할 작업 메시지와 완료 상태를 전달·보관합니다.
- **Celery Worker**: FastAPI와 별도의 프로세스로 실제 파일 생성 작업을 처리합니다.
- **PostgreSQL**: 검사·동작인식의 구조화된 기록이 계속 보관되는 원본 저장소입니다.

Celery를 설치했다고 **AI 추론 자체가 빨라지는 것은 아닙니다.** 오늘은 결과 내보내기 작업을 웹 요청과 분리하여 FastAPI가 오래 기다리지 않도록 합니다. 실제 AI 모델 추론은 4일차에 연결합니다.

### 3.7. 이번 수업의 비동기 처리와 파일 보관 기준

작업을 등록하면 FastAPI는 **HTTP 202와 작업 ID**를 반환합니다. 파일 생성이 완료되는 것은 Celery Worker의 실행 결과를 확인한 뒤입니다. `PENDING`은 아직 기다리는 작업일 수도 있고, 존재하지 않거나 결과가 만료된 작업일 수도 있으므로 그것만으로 Worker가 정상이라고 판단하지 않습니다.

Redis의 작업 결과 보관기간은 오늘 예제에서 **1시간**입니다. 반면 내보낸 파일은 `data/day03/exports/작업ID/`에 남습니다. 파일은 PostgreSQL 테이블 안에 넣지 않고 로컬 디스크에 저장합니다. 다른 컴퓨터의 Worker를 사용할 경우에는 두 프로세스가 같은 파일 저장 위치에 접근할 수 있도록 별도 구성이 필요하지만, 오늘은 **한 대의 개발 PC에서 모두 실행**합니다.

**보안:** Redis와 PostgreSQL은 오늘 로컬 개발용으로 사용합니다. Redis의 포트 6379를 외부 인터넷에 공개하지 않고, 비밀번호·원본 검사 자료·생성된 데이터 파일을 공개 GitHub에 올리지 않습니다. Docker는 7일차에 사용합니다.

---

## 4. 오늘 사용할 기술 스택

| 기술 | 오늘 실습에서 하는 일 |
|---|---|
| Python `.venv` | 1·2일차와 동일한 개발환경에서 계속 작업 |
| SQLAlchemy 2.x · PostgreSQL | 기존 AI 결과를 조회하고 스냅샷 생성의 입력으로 사용 |
| JSON / JSONL | 이벤트를 한 줄씩 기록하여 보존 |
| PyArrow / Parquet | `Silver` 분석용 데이터 파일 생성·검증 |
| Redis | Celery 작업 전달과 완료 상태 저장 |
| Celery | 별도 Worker 프로세스에서 파일 생성 작업 실행 |
| FastAPI · Pydantic | Export 요청과 작업 상태 조회 API 제공 |
| HTML · CSS · JavaScript `fetch()` | 작업 요청·상태 확인 화면 구성 |
| Pytest · HTTPX | 기존 기능 유지와 데이터 변환·비동기 API 검사 |
| Git | 3일차 변경 파일을 기존 저장소에 커밋 |

---

# 5. 처음부터 끝까지 따라 하는 상세 실습

## PART 1. 2일차 프로젝트와 기존 기록 확인

### 실습 1. 프로젝트와 `.venv`를 다시 엽니다

WSL Ubuntu 터미널에서 다음을 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
pwd
which python
```

`pwd`의 마지막은 `subject13_ai_platform`이어야 하고, `which python`에는 이 프로젝트의 `.venv/bin/python`이 표시되어야 합니다. 교과 9나 교과 10의 가상환경을 사용하지 않습니다.

먼저 PostgreSQL 서버를 확인합니다.

```bash
pg_isready -h 127.0.0.1 -p 5432
```

`accepting connections`가 나오면 다음으로 진행합니다. 서버가 꺼져 있다면 2일차와 같이 `sudo service postgresql start`로 시작합니다.

전날 사용한 DB API의 정상 동작도 확인합니다.

```bash
python -m pytest -q
```

1일차 테스트 12개와 2일차 테스트 7개가 남아 있다면 **19개가 통과**해야 합니다. 아직 3일차 코드를 작성하기 전이므로 오늘의 추가 테스트는 포함되지 않습니다.

### 실습 2. DB에 저장된 AI 결과가 있는지 확인합니다

PostgreSQL에 2일차 이벤트가 실제 저장되어 있는지 확인합니다.

```bash
psql -h 127.0.0.1 -U subject13_user -d subject13_platform
```

접속한 다음 SQL을 실행합니다.

```sql
SELECT source_mode, task, decision, COUNT(*) AS event_count
FROM inference_events
GROUP BY source_mode, task, decision
ORDER BY source_mode, task, decision;
```

교과 9의 `inspection`, 교과 10의 `action` 기록이 나와야 합니다. `recorded_result`는 이전 교과에서 저장한 실제 결과이고, `manual_entry`는 2일차 CRUD 연습 데이터입니다. **오늘의 내보내기에서는 `recorded_result`만 사용합니다.** 확인을 마치면 `\q`로 종료합니다.

만약 `recorded_result`가 하나도 없다면 2일차 DB 이관을 다시 실행합니다.

```bash
python -m scripts.03_import_records_to_db
```

이 명령은 `event_id`를 확인하여 이전에 이관된 기록을 중복 저장하지 않습니다.

### 실습 3. 오늘 새로 만들 파일과 수정할 파일을 확인합니다

오늘 사용할 폴더를 추가합니다.

```bash
mkdir -p data/day03/exports docs
```

오늘 **새로 작성하는 파일**은 다음과 같습니다.

```text
subject13_ai_platform/
├── app/
│   ├── data_pipeline.py            # DB → Bronze/Silver/Gold
│   ├── celery_app.py               # Redis·Celery 설정
│   ├── worker_tasks.py             # 백그라운드 실행 함수
│   └── pipeline_api.py             # 작업 요청·조회 API
├── scripts/
│   ├── 04_check_day03_inputs.py    # PostgreSQL·Redis 확인
│   ├── 05_export_once.py           # 동기 내보내기 검사
│   └── 06_inspect_export.py        # JSONL·Parquet·Gold 확인
├── web/
│   └── pipeline.html               # 3일차 비동기 작업 화면
├── tests/
│   └── test_day03.py               # 새 테스트 7개
├── docs/
│   └── day03_data_design.md        # 데이터·비동기 구조 문서
└── data/day03/exports/             # 생성 결과 파일(Git 제외)
```

**기존 파일 중 수정할 것은 다음뿐입니다.**

```text
requirements.txt         ← 3일차 패키지 세 줄 추가
.env.example             ← Redis 설정 한 줄 추가
.env                     ← Redis 설정 한 줄 추가(Git 제외)
app/main.py              ← Pipeline Router 두 줄 추가
web/db.html              ← 3일차 페이지 이동 링크 추가
.gitignore               ← Export 결과 폴더 제외
README.md                ← 3일차 실행 방법 추가
docs/architecture.md     ← 3일차 완성 기능 한 문단 추가
```

`app/db.py`, `app/models.py`, `app/repository.py`, `app/db_api.py`, `app/schemas.py`와 기존 테스트 파일은 수정하지 않습니다. 현재의 PostgreSQL 테이블 세 개에도 새 컬럼을 추가하지 않습니다.

---

## PART 2. 데이터 구조와 실행환경 준비

### 실습 4. Bronze·Silver·Gold의 파일 구조를 먼저 설계합니다

오늘 다룰 데이터 흐름을 파일 단위로 나타내면 다음과 같습니다.

```text
PostgreSQL / inference_events
                ↓
        row_to_event() 재사용
                ↓
        ┌───────┼────────┐
        ▼       ▼        ▼
     Bronze   Silver     Gold
      JSONL   Parquet     JSON
        └───────┼────────┘
                ↓
         manifest.json
       (파일 위치·건수·작업 ID)
```

예를 들어 `inspection` 이벤트 하나를 내보낼 때 Bronze에서는 공통 이벤트와 이미지명 등의 정보를 함께 보관합니다. Silver에서는 `task`, `confidence`, `decision` 등을 일정한 열 이름으로 정리합니다. Gold에서는 전체 결과의 `OK/NG/UNKNOWN` 건수를 셉니다.

파일은 내보내기 요청마다 **새로운 작업 ID 폴더**에 저장됩니다.

```text
data/day03/exports/
└── 작업_UUID/
    ├── bronze_events.jsonl
    ├── silver_events.parquet
    ├── gold_summary.json
    └── manifest.json
```

`manifest.json`은 다른 파일이 정상적으로 저장된 다음 **마지막에 생성**합니다. 나중에 작업 결과를 확인할 때 어떤 파일이 만들어졌고, 몇 건을 처리했는지 연결할 수 있습니다.

### 실습 5. `requirements.txt`에 새 패키지를 추가합니다

1·2일차에 작성한 패키지 목록을 지우지 않습니다. **맨 아래에 다음 세 줄만 추가**합니다.

```text
celery>=5.4,<6
redis>=5,<7
pyarrow>=18,<25
```

패키지별 역할은 다음과 같습니다.

- `celery`: 웹 요청과 별개로 Worker 프로세스에서 작업을 처리합니다.
- `redis`: Python에서 Redis에 연결해 작업 등록·상태 확인을 수행합니다.
- `pyarrow`: JSONL 다음 단계의 Parquet 파일을 만들고 읽습니다.

기존 `.venv`에서 다음을 실행합니다.

```bash
python -m pip install -r requirements.txt
python -m pip show celery redis pyarrow
```

설치 결과에서 세 패키지의 이름과 버전이 보이는지 확인합니다. `pyarrow`는 바이너리 패키지이므로 인터넷이 제한된 실습실에서는 **미리 제공된 내부 패키지 저장소나 Wheel 파일**이 필요할 수 있습니다. 해당 환경에서 지원되는 Python 버전과 Wheel이 일치해야 합니다.

### 실습 6. Redis를 설치·실행합니다

오늘은 Docker 없이 WSL Ubuntu에서 Redis 서버를 실행합니다. 먼저 설치 여부를 확인합니다.

```bash
redis-server --version
```

아직 설치되지 않았다면 다음 명령을 사용합니다.

```bash
sudo apt update
sudo apt install -y redis-server redis-tools
```

Redis를 시작합니다.

```bash
sudo service redis-server start
redis-cli ping
```

정상이라면 다음 한 단어가 출력됩니다.

```text
PONG
```

`PONG`은 **Redis 서버가 현재 명령에 응답했다**는 뜻입니다. 이것만으로 Celery Worker까지 실행되고 있다는 뜻은 아닙니다. Worker는 뒤에서 별도로 시작합니다.

기관에서 제공하는 Redis를 사용한다면 로컬 Redis를 중복 설치하지 않고 안내받은 서버를 사용합니다. 다만 Redis 서버에 외부 접속을 허용하거나 보안 설정을 임의로 변경하지 않습니다.

### 실습 7. 2일차 `.env`에 Redis 설정을 추가합니다

2일차 `.env`에 있는 PostgreSQL 계정·비밀번호를 지우지 않습니다. **`.env.example`과 `.env` 두 파일의 맨 아래에 다음 한 줄을 추가**합니다.

```dotenv
REDIS_URL=redis://127.0.0.1:6379/0
```

- `127.0.0.1`: 이 개발 PC에서 실행되는 Redis를 사용합니다.
- `6379`: Redis의 기본 포트입니다.
- `/0`: 오늘 실습에서 사용할 Redis 논리 데이터베이스 번호입니다.

기관 Redis의 주소·인증정보가 다르면 실제 값에 맞춰 `.env`만 수정합니다. **암호가 포함된 Redis 주소는 `.env.example`에 기록하지 않습니다.** `.env`는 Git에서 제외된 상태를 유지합니다.

### 실습 8. PostgreSQL과 Redis의 준비 상태 검사 코드를 작성합니다

앞에서 두 서버를 각각 확인했습니다. 이제 Python 코드에서도 같은 환경에 접근할 수 있는지 검사합니다. 이 파일은 AI 모델을 실행하거나 데이터를 내보내지 않습니다.

**의사코드**

```text
.env와 기존 SQLAlchemy 연결 사용
       ↓
PostgreSQL SELECT 1 실행
       ↓
inference_events의 recorded_result 건수 확인
       ↓
Redis에 PING 보내기
       ↓
모두 정상이라면 READY 출력
```

**파일: `scripts/04_check_day03_inputs.py`**

```python
"""3일차 실습 시작 전 기존 프로젝트·DB·Redis 준비 상태를 확인합니다."""
import os
from pathlib import Path

import redis
from dotenv import load_dotenv
from sqlalchemy import func, select, text

from app.db import SessionLocal
from app.models import InferenceEventRow

ROOT = Path(__file__).resolve().parents[1]
load_dotenv(ROOT / '.env')


def main():
    with SessionLocal() as db:
        db.execute(text('SELECT 1'))
        total = db.scalar(select(func.count()).select_from(InferenceEventRow)) or 0
        recorded = db.scalar(select(func.count()).select_from(InferenceEventRow).where(
            InferenceEventRow.source_mode == 'recorded_result')) or 0
    print('PostgreSQL 연결: OK')
    print(f'DB 전체 기록: {total}건 / 기존 AI 기록: {recorded}건')
    if recorded == 0:
        raise RuntimeError('기존 AI 기록이 없습니다. 2일차 DB 이관부터 확인하세요.')
    url = os.getenv('REDIS_URL', 'redis://127.0.0.1:6379/0')
    client = redis.Redis.from_url(url, socket_connect_timeout=2, socket_timeout=2)
    try:
        if not client.ping():
            raise RuntimeError('Redis 응답이 없습니다.')
    finally:
        client.close()
    print('Redis PING: PONG')
    print('3일차 DB·Redis 입력 상태: READY')


if __name__ == '__main__':
    main()
```

프로젝트 최상위에서 실행합니다.

```bash
python -m scripts.04_check_day03_inputs
```

결과 예시:

```text
PostgreSQL 연결: OK
DB 전체 기록: 11건 / 기존 AI 기록: 11건
Redis PING: PONG
3일차 DB·Redis 입력 상태: READY
```

위 `11건`은 이전 교과 결과가 교과 9에서 5건, 교과 10에서 6건인 경우의 예입니다. **학생마다 건수가 달라도 `recorded_result`가 1건 이상이고 연결 검사가 통과하면 다음으로 진행**합니다.

---

## PART 3. JSONL·Parquet·Gold 저장 프로그램 구현

### 실습 9. 실제 PostgreSQL 기록을 내보내는 코드를 작성합니다

이번에는 PostgreSQL에서 이벤트를 읽어 **세 종류의 데이터 파일**로 저장하는 핵심 코드를 작성합니다. 2일차의 `InferenceEventRow`와 `row_to_event()`를 다시 사용하므로 이전 결과 변환이나 CRUD 코드는 수정하지 않습니다.

**실행 흐름을 의사코드로 먼저 확인합니다.**

```text
요청한 작업 ID가 UUID 형식인지 검사
         ↓
PostgreSQL의 기존 AI 이벤트(recorded_result)만 읽기
         ↓
입력 출처·처리 모듈을 관계(FK)로 함께 조회
         ↓
원본 이벤트를 1일차 공통 JSON으로 복원
         ↓
Bronze: 이벤트 + 이미지·영상 참조 메타데이터 → JSONL
         ↓
Silver: 공통 필드들을 자료형이 있는 컬럼으로 구성 → Parquet
         ↓
Parquet 파일을 다시 읽어 행 수 검사
         ↓
Gold: 교과별·판정별 개수 → JSON
         ↓
마지막에 manifest.json 생성
         ↓
작업 ID / 파일 상대경로 / 저장 건수를 반환
```

**먼저 이해할 핵심 규칙**

`source_mode='recorded_result'`만 선택하므로 2일차에 직접 만든 `manual_entry` 연습 결과는 AI가 추론한 결과인 것처럼 내보내지 않습니다. `confidence`가 없으면 `None/null`을 유지하고, `reasons`와 `details`는 JSON 문자열로 직렬화하여 Silver에서도 의미를 잃지 않도록 합니다. 원본 `recorded_at_raw`도 그대로 보존하며, 날짜를 임의로 한국시간이나 UTC로 가정하지 않습니다.

`limit`는 처음 실습에서 처리할 최대 기록 수입니다. 이번 예제에서는 1~1000건으로 제한하여 파일 저장 흐름을 이해하는 데 집중합니다. 데이터가 1000건보다 많더라도 이 파일은 **전체를 자동으로 처리한다고 주장하지 않습니다.** 이번 수업의 `limit`은 교육용 일부 기록 스냅샷을 만드는 제한입니다.

**파일: `app/data_pipeline.py`**

```python
"""3일차: DB 기록을 JSONL(Bronze)·Parquet(Silver)·JSON(Gold)로 내보냅니다."""
import json
from collections import Counter
from datetime import datetime, timezone
from pathlib import Path
from uuid import UUID

import pyarrow as pa
import pyarrow.parquet as pq
from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from app.models import InferenceEventRow
from app.repository import row_to_event

ROOT = Path(__file__).resolve().parents[1]
EXPORT_ROOT = ROOT / 'data' / 'day03' / 'exports'


def silver_schema() -> pa.Schema:
    """교과 9·10 공통 이벤트를 열 기반 분석용 스키마로 표현합니다."""
    return pa.schema([
        pa.field('event_id', pa.string(), nullable=False),
        pa.field('task', pa.string(), nullable=False),
        pa.field('source_mode', pa.string(), nullable=False),
        pa.field('source_project', pa.string(), nullable=False),
        pa.field('source_path', pa.string(), nullable=False),
        pa.field('recorded_at_raw', pa.string(), nullable=False),
        pa.field('recorded_at_utc', pa.string()),
        pa.field('input_ref', pa.string(), nullable=False),
        pa.field('asset_kind', pa.string(), nullable=False),
        pa.field('processor_id', pa.string(), nullable=False),
        pa.field('processor_version', pa.string()),
        pa.field('predicted_label', pa.string()),
        pa.field('expected_label', pa.string()),
        pa.field('confidence', pa.float64()),
        pa.field('decision', pa.string(), nullable=False),
        pa.field('reasons_json', pa.string(), nullable=False),
        pa.field('details_json', pa.string(), nullable=False),
    ])


def export_event_snapshot(db: Session, job_id: str, task_filter: str | None = None, limit: int = 1000) -> dict:
    """독립 실행 가능한 동기 함수입니다. Celery 작업에서 이 함수를 호출합니다."""
    try:
        safe_job_id = str(UUID(job_id))
    except (ValueError, TypeError, AttributeError) as error:
        raise ValueError('job_id는 UUID 형식이어야 합니다.') from error
    if task_filter not in (None, 'inspection', 'action'):
        raise ValueError('task_filter는 inspection/action/None 중 하나여야 합니다.')
    if not (1 <= limit <= 1000):
        raise ValueError('limit는 1~1000 범위여야 합니다.')

    stmt = (
        select(InferenceEventRow)
        .options(selectinload(InferenceEventRow.source), selectinload(InferenceEventRow.model))
        .where(InferenceEventRow.source_mode == 'recorded_result')
        .order_by(InferenceEventRow.id.asc())
        .limit(limit)
    )
    if task_filter:
        stmt = stmt.where(InferenceEventRow.task == task_filter)
    records = db.scalars(stmt).all()
    if not records:
        raise ValueError('내보낼 기존 AI 결과가 없습니다. 2일차 DB 이관 결과를 확인하세요.')

    output = EXPORT_ROOT / safe_job_id
    output.mkdir(parents=True, exist_ok=True)
    exported_at = datetime.now(timezone.utc).isoformat()
    bronze_file = output / 'bronze_events.jsonl'
    silver_file = output / 'silver_events.parquet'
    gold_file = output / 'gold_summary.json'
    manifest_file = output / 'manifest.json'

    silver_rows: list[dict] = []
    with bronze_file.open('w', encoding='utf-8') as stream:
        for record in records:
            common = row_to_event(record).model_dump(mode='json')
            # 파일 자체의 존재 여부는 검사하지 않습니다. 4일차에 실제 영상과 연결합니다.
            asset_kind = {'image_file': 'image', 'video_file': 'video'}.get(
                record.source.source_type, 'unknown'
            )
            bronze = {
                'snapshot_version': 'day03-v1',
                'exported_at_utc': exported_at,
                'event': common,
                'asset_metadata': {
                    'input_ref': record.input_ref,
                    'asset_kind': asset_kind,
                    'asset_file_verified': False,
                },
            }
            stream.write(json.dumps(bronze, ensure_ascii=False, allow_nan=False) + '\n')
            silver_rows.append({
                'event_id': record.event_id,
                'task': record.task,
                'source_mode': record.source_mode,
                'source_project': record.source_project,
                'source_path': record.source_path,
                'recorded_at_raw': record.recorded_at_raw,
                'recorded_at_utc': record.recorded_at_utc.isoformat() if record.recorded_at_utc else None,
                'input_ref': record.input_ref,
                'asset_kind': asset_kind,
                'processor_id': record.model.processor_id,
                'processor_version': record.model.version,
                'predicted_label': record.predicted_label,
                'expected_label': record.expected_label,
                'confidence': record.confidence,
                'decision': record.decision,
                'reasons_json': json.dumps(record.reasons, ensure_ascii=False),
                'details_json': json.dumps(record.details, ensure_ascii=False, allow_nan=False),
            })

    table = pa.Table.from_pylist(silver_rows, schema=silver_schema())
    pq.write_table(table, silver_file, compression='snappy')
    verified_table = pq.read_table(silver_file)
    if verified_table.num_rows != len(records):
        raise RuntimeError('Parquet 저장 후 행 수 검증에 실패했습니다.')

    task_counts = Counter(item['task'] for item in silver_rows)
    decision_counts = Counter(item['decision'] for item in silver_rows)
    grouped = Counter((item['task'], item['decision']) for item in silver_rows)
    gold = {
        'snapshot_version': 'day03-v1',
        'generated_at_utc': exported_at,
        'source': '2일차 PostgreSQL 기존 저장 이벤트',
        'total': len(silver_rows),
        'by_task': {key: task_counts[key] for key in ('inspection', 'action')},
        'by_decision': {key: decision_counts[key] for key in ('OK', 'NG', 'UNKNOWN')},
        'by_task_decision': {
            key: {decision: grouped[(key, decision)] for decision in ('OK', 'NG', 'UNKNOWN')}
            for key in ('inspection', 'action')
        },
        'confidence_missing': sum(row['confidence'] is None for row in silver_rows),
        'note': '저장된 교육용 기록의 개수이며, 실시간 검사 성능 또는 정확도가 아닙니다.',
    }
    gold_file.write_text(json.dumps(gold, ensure_ascii=False, indent=2), encoding='utf-8')
    rel = lambda filename: (output / filename).relative_to(ROOT).as_posix()
    artifacts = {
        'bronze': rel('bronze_events.jsonl'),
        'silver': rel('silver_events.parquet'),
        'gold': rel('gold_summary.json'),
        'manifest': rel('manifest.json'),
    }
    manifest = {
        'job_id': safe_job_id,
        'completed_at_utc': datetime.now(timezone.utc).isoformat(),
        'row_count': len(silver_rows),
        'task_filter': task_filter,
        'limit': limit,
        'schema_version': 'day03-v1',
        'artifacts': artifacts,
    }
    # 모든 데이터 파일을 성공적으로 저장한 뒤 완료 표시인 manifest를 마지막에 작성합니다.
    manifest_file.write_text(json.dumps(manifest, ensure_ascii=False, indent=2), encoding='utf-8')
    return manifest
```

**주요 코드 이해하기**

`silver_schema()`에서는 Parquet의 열 이름과 자료형을 명시합니다. 이때 신뢰도는 `float64`이며 값이 없는 OCR 이벤트의 `confidence`는 그대로 `null`이 될 수 있습니다.

`selectinload()`는 이벤트의 입력 출처(`source`)와 처리 모듈(`model`)을 함께 가져옵니다. 이 정보가 있어야 이미지·영상 종류를 **파일 참조 메타데이터**로 기록할 수 있습니다. 다만 원본 이미지·영상 파일을 직접 열어 검사하는 것은 아닙니다.

`uuid.UUID()`를 사용하는 이유는 작업 ID를 경로 이름으로 사용할 때 임의의 파일 경로가 입력되지 않도록 제한하기 위해서입니다. `manifest.json`은 출력 파일의 상대경로만 기록하고, DB 접속 비밀번호나 시스템의 절대경로를 기록하지 않습니다.

`pq.write_table()`은 Silver 데이터를 Parquet으로 저장하고, `pq.read_table()`은 저장한 결과를 다시 열어 행 수를 확인합니다. 최종 Gold 집계에서는 `UNKNOWN`을 NG로 합치지 않습니다.

### 실습 10. 비동기 처리 전에 동기 내보내기를 먼저 검증합니다

Celery를 붙이기 전에 **현재 작성한 데이터 변환 코드가 단독으로 정상 실행되는지 먼저 확인**합니다. 이 단계가 성공하면 이후 문제가 생겨도 데이터 처리 오류와 Redis·Worker 연결 오류를 구분할 수 있습니다.

**의사코드**

```text
새 UUID 생성
      ↓
2일차 PostgreSQL Session 생성
      ↓
export_event_snapshot() 직접 호출
      ↓
파일 4개 생성
      ↓
출력 경로와 행 수 확인
```

**파일: `scripts/05_export_once.py`**

```python
"""Celery 연결 전에 데이터를 동기 방식으로 내보내 코드·형식 오류를 확인합니다."""
import json
from uuid import uuid4

from app.data_pipeline import export_event_snapshot
from app.db import SessionLocal


def main():
    with SessionLocal() as db:
        result = export_event_snapshot(db, job_id=str(uuid4()), limit=100)
    print(json.dumps(result, ensure_ascii=False, indent=2))
    print('동기 내보내기 완료: JSONL / Parquet / Gold / Manifest')


if __name__ == '__main__':
    main()
```

다음 명령으로 동기 내보내기를 실행합니다.

```bash
python -m scripts.05_export_once
```

성공하면 다음과 비슷하게 출력됩니다. UUID와 기록 수는 실행할 때마다 달라집니다.

```json
{
  "job_id": "a3c704f0-a9fb-4b24-8fc7-cf7e8f20dc91",
  "row_count": 11,
  "task_filter": null,
  "limit": 100,
  "schema_version": "day03-v1",
  "artifacts": {
    "bronze": "data/day03/exports/a3c704f0-a9fb-4b24-8fc7-cf7e8f20dc91/bronze_events.jsonl",
    "silver": "data/day03/exports/a3c704f0-a9fb-4b24-8fc7-cf7e8f20dc91/silver_events.parquet",
    "gold": "data/day03/exports/a3c704f0-a9fb-4b24-8fc7-cf7e8f20dc91/gold_summary.json",
    "manifest": "data/day03/exports/a3c704f0-a9fb-4b24-8fc7-cf7e8f20dc91/manifest.json"
  }
}
```

위 JSON은 **구조를 보여 주는 예시**이며 `completed_at_utc`도 실제 응답에는 포함됩니다. `row_count`는 현재 저장된 기록 수와 `limit`에 따라 달라집니다. 지금은 Celery가 아니라 터미널에서 동기 방식으로 함수를 직접 실행한 것입니다.

### 실습 11. JSONL·Parquet·Gold 파일을 직접 확인합니다

위 실행으로 생성된 작업 폴더를 확인합니다.

```bash
find data/day03/exports -maxdepth 2 -type f | sort
```

Bronze JSONL의 첫 줄을 직접 확인하려면 가장 최근 작업의 경로를 찾아야 합니다. 아래 실습에서는 **가장 최근에 완성된 `manifest.json`을 기준**으로 자동 확인하는 프로그램을 작성합니다.

**의사코드**

```text
최근 완료 manifest.json 검색
        ↓
해당 폴더의 Bronze 첫 줄 읽기
        ↓
Silver Parquet 열 이름과 행 수 확인
        ↓
Gold 집계 JSON 열기
        ↓
Manifest 행 수 = Parquet 행 수 = Gold 전체 건수 비교
        ↓
일치하면 PASS
```

**파일: `scripts/06_inspect_export.py`**

```python
"""가장 최근 완료된 내보내기의 파일 내용·행 수를 확인합니다."""
import json
from pathlib import Path

import pyarrow.parquet as pq

from app.data_pipeline import EXPORT_ROOT


def main():
    candidates = sorted(EXPORT_ROOT.glob('*/manifest.json'), key=lambda path: path.stat().st_mtime)
    if not candidates:
        raise FileNotFoundError('manifest.json이 없습니다. 먼저 내보내기를 실행하세요.')
    path = candidates[-1]
    manifest = json.loads(path.read_text(encoding='utf-8'))
    root = path.parent
    with (root / 'bronze_events.jsonl').open(encoding='utf-8') as stream:
        first = json.loads(next(stream))
    table = pq.read_table(root / 'silver_events.parquet')
    gold = json.loads((root / 'gold_summary.json').read_text(encoding='utf-8'))
    print('Export ID:', manifest['job_id'])
    print('Bronze 첫 이벤트:', first['event']['event_id'], first['asset_metadata'])
    print('Silver Parquet 행 수:', table.num_rows)
    print('Silver 열 이름:', table.column_names)
    print('Gold 집계:', json.dumps(gold, ensure_ascii=False, indent=2))
    assert manifest['row_count'] == table.num_rows == gold['total']
    print('검증 결과: PASS (Manifest = Parquet = Gold 행 수)')


if __name__ == '__main__':
    main()
```

실행합니다.

```bash
python -m scripts.06_inspect_export
```

다음과 같은 정보가 나와야 합니다.

```text
Export ID: ...
Bronze 첫 이벤트: inspection:... {'input_ref': '...', 'asset_kind': 'image', 'asset_file_verified': False}
Silver Parquet 행 수: 11
Silver 열 이름: ['event_id', 'task', ...]
Gold 집계: {...}
검증 결과: PASS (Manifest = Parquet = Gold 행 수)
```

이제 **DB 조회 → JSONL·Parquet·Gold 파일 저장**까지 완료되었습니다. 다음 PART에서 이 함수를 Celery Worker로 옮겨 비동기로 실행합니다.

---

## PART 4. Redis·Celery로 파일 생성 작업 분리

### 실습 12. Celery 앱 설정 파일을 작성합니다

Celery를 사용하려면 Redis 서버의 주소와 결과 처리 규칙을 알아야 합니다. 이 설정을 FastAPI 코드 안에 직접 적는 대신 `app/celery_app.py`에 분리합니다.

**의사코드**

```text
기존 .env에서 REDIS_URL 확인
         ↓
Celery 앱 생성
         ↓
Redis를 Broker로 설정
         ↓
Redis를 작업 결과 저장소로 설정
         ↓
JSON 메시지만 허용
         ↓
작업 시작 상태 추적·결과 보관기간 설정
```

**파일: `app/celery_app.py`**

```python
"""3일차: Redis를 broker/result backend로 사용하는 Celery 설정."""
import os
from pathlib import Path

from celery import Celery
from dotenv import load_dotenv

ROOT = Path(__file__).resolve().parents[1]
load_dotenv(ROOT / '.env')
REDIS_URL = os.getenv('REDIS_URL', 'redis://127.0.0.1:6379/0')

celery_app = Celery(
    'subject13',
    broker=REDIS_URL,
    backend=REDIS_URL,
    include=['app.worker_tasks'],
)
celery_app.conf.update(
    task_serializer='json',
    result_serializer='json',
    accept_content=['json'],
    task_track_started=True,
    result_expires=3600,
    broker_connection_retry_on_startup=True,
    worker_prefetch_multiplier=1,
)
```

**코드 이해하기**

`broker=REDIS_URL`은 작업 메시지를 Redis를 통해 전달한다는 뜻이고, `backend=REDIS_URL`은 Worker의 처리 결과를 Redis에 저장해 FastAPI가 상태를 확인하도록 한다는 뜻입니다. 오늘은 같은 Redis 서버를 두 역할에 사용합니다.

`task_track_started=True`는 Worker가 실제 실행하기 시작한 작업을 `STARTED`로 표시하도록 합니다. `result_expires=3600`은 완료 결과를 무한히 Redis에 남기지 않고 일정 시간이 지나면 만료되도록 설정한 것입니다.

### 실습 13. Worker가 실행할 비동기 작업을 작성합니다

앞에서 작성한 `export_event_snapshot()`는 동기 함수입니다. 이번에는 Celery Worker가 이 함수를 실행하도록 작업을 등록합니다.

**의사코드**

```text
Celery가 작업 메시지 받기
       ↓
Celery 작업 ID 가져오기
       ↓
Worker 안에서 PostgreSQL Session 새로 열기
       ↓
동기 Export 함수 실행
       ↓
결과 Manifest 반환
       ↓
Session 종료 후 Celery 결과를 Redis에 기록
```

특히 **FastAPI의 DB Session 객체를 Worker에 전달하지 않습니다.** Session은 작업 메시지에 넣을 데이터가 아니라 DB 연결과 트랜잭션을 다루는 객체이기 때문입니다. 오늘은 작업의 `task_filter`와 `limit`처럼 JSON으로 직렬화할 수 있는 값만 Redis 메시지로 전달합니다.

**파일: `app/worker_tasks.py`**

```python
"""3일차: API 프로세스와 별도 프로세스에서 실행되는 작업."""
from app.celery_app import celery_app
from app.data_pipeline import export_event_snapshot
from app.db import SessionLocal


@celery_app.task(bind=True, name='day03.export_events')
def export_events_task(self, task_filter: str | None = None, limit: int = 1000) -> dict:
    job_id = self.request.id
    if not job_id:
        raise RuntimeError('Celery 작업 ID를 받지 못했습니다.')
    # FastAPI 요청 Session을 넘기지 않고 Worker가 자신의 DB Session을 사용합니다.
    with SessionLocal() as db:
        return export_event_snapshot(db, job_id=job_id, task_filter=task_filter, limit=limit)
```

`@celery_app.task(bind=True, ...)`는 함수에 Celery 작업 정보를 연결합니다. `self.request.id`는 내보내기 결과 폴더 이름으로 사용할 UUID이며, FastAPI는 나중에 같은 ID로 작업 상태를 조회합니다.

### 실습 14. Celery Worker를 별도 터미널에서 실행합니다

PostgreSQL과 Redis가 실행 중인지 다시 확인합니다.

```bash
pg_isready -h 127.0.0.1 -p 5432
redis-cli ping
```

정상이면 **새 WSL 터미널**을 열고 같은 프로젝트로 이동합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
celery -A app.celery_app:celery_app worker --loglevel=info --pool=solo
```

`--pool=solo`는 오늘의 실습에서 동작 과정을 쉽게 확인하기 위한 단일 실행 프로세스 방식입니다. Worker 터미널에는 다음과 비슷한 정보가 표시됩니다.

```text
[tasks]
  . day03.export_events

celery@... ready.
```

`day03.export_events`가 작업 목록에 나타나고 `ready`가 보이면 작업 수신 준비가 된 것입니다. **이 터미널은 닫지 않습니다.** 작업을 요청하면 여기에서 실제 파일 생성 로그를 확인할 수 있습니다.

Redis 서버 시작과 Worker 실행은 서로 다른 과정입니다. `redis-cli ping`이 `PONG`이어도 Worker 터미널을 종료하면 파일 생성 작업이 진행되지 않습니다.

---

## PART 5. FastAPI와 비동기 작업 화면 연결

### 실습 15. FastAPI용 작업 등록·상태 조회 API를 작성합니다

1일차에는 기존 JSON을 조회하는 API, 2일차에는 PostgreSQL CRUD API를 작성했습니다. 3일차에는 **기존 두 API를 유지**하면서 파일 생성 작업을 요청하는 새로운 경로를 추가합니다.

```text
기존 1일차 API
GET /api/v1/events
→ 이전 JSON 기록 조회

기존 2일차 API
GET /api/v1/db/events
→ PostgreSQL 기록 조회

새 3일차 API
GET  /api/v1/pipeline/health
→ PostgreSQL·Redis 연결 확인

POST /api/v1/pipeline/exports
→ Celery 작업 등록, 작업 ID 반환

GET  /api/v1/pipeline/jobs/{job_id}
→ PENDING / STARTED / SUCCESS / FAILURE 상태 조회
```

**의사코드**

```text
브라우저에서 내보내기 종류·최대 건수 선택
          ↓
FastAPI가 Pydantic으로 입력 검사
          ↓
export_events_task.delay()로 Redis에 작업 등록
          ↓
FastAPI는 바로 HTTP 202와 작업 ID 반환
          ↓
Celery Worker가 DB 조회와 파일 저장 수행
          ↓
브라우저가 job_id로 작업 상태 다시 요청
          ↓
완료하면 Manifest·파일 상대경로 표시
```

**중요:** 작업 등록의 HTTP `202 Accepted`는 결과 파일 저장 성공을 의미하지 않습니다. 반드시 상태 조회에서 `SUCCESS`와 `result`의 파일 정보를 확인해야 합니다.

**파일: `app/pipeline_api.py`**

```python
"""3일차: 비동기 Export 작업 요청·상태 확인 API."""
import os
from pathlib import Path

import redis
from celery.result import AsyncResult
from dotenv import load_dotenv
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel, Field
from sqlalchemy import text
from sqlalchemy.orm import Session
from fastapi import Depends
from typing import Literal

from app.celery_app import celery_app
from app.db import get_db
from app.worker_tasks import export_events_task
from fastapi.responses import FileResponse

ROOT = Path(__file__).resolve().parents[1]
load_dotenv(ROOT / '.env')
router = APIRouter()
PIPELINE_PAGE = ROOT / 'web' / 'pipeline.html'


class ExportRequest(BaseModel):
    task: Literal['inspection', 'action'] | None = None
    limit: int = Field(default=100, ge=1, le=1000)


@router.get('/pipeline')
def pipeline_page():
    return FileResponse(PIPELINE_PAGE)


@router.get('/api/v1/pipeline/health')
def pipeline_health(db: Session = Depends(get_db)):
    db_ready = False
    redis_ready = False
    try:
        db.execute(text('SELECT 1'))
        db_ready = True
    except Exception:
        pass
    try:
        url = os.getenv('REDIS_URL', 'redis://127.0.0.1:6379/0')
        client = redis.Redis.from_url(url, socket_connect_timeout=1, socket_timeout=1)
        try:
            redis_ready = bool(client.ping())
        finally:
            client.close()
    except redis.RedisError:
        pass
    return {'status': 'ok' if db_ready and redis_ready else 'degraded',
            'postgresql': db_ready, 'redis': redis_ready,
            'worker': '이 API는 Worker 실행 여부까지 확인하지 않습니다.'}


@router.post('/api/v1/pipeline/exports', status_code=202)
def request_export(payload: ExportRequest):
    try:
        job = export_events_task.delay(task_filter=payload.task, limit=payload.limit)
    except Exception as error:
        raise HTTPException(status_code=503, detail='Redis 연결 또는 작업 등록을 확인하세요.') from error
    return {'job_id': job.id, 'status': 'PENDING',
            'status_url': f'/api/v1/pipeline/jobs/{job.id}'}


@router.get('/api/v1/pipeline/jobs/{job_id}')
def job_status(job_id: str):
    from uuid import UUID
    try:
        UUID(job_id)
    except ValueError as error:
        raise HTTPException(status_code=422, detail='작업 ID는 UUID 형식이어야 합니다.') from error
    try:
        result = AsyncResult(job_id, app=celery_app)
        state = result.state
        manifest = result.result if state == 'SUCCESS' and isinstance(result.result, dict) else None
    except Exception as error:
        raise HTTPException(status_code=503, detail='Redis 작업 상태를 조회할 수 없습니다.') from error
    return {'job_id': job_id, 'status': state, 'result': manifest,
            'note': 'PENDING은 대기 또는 결과 정보가 없는 상태를 모두 포함할 수 있습니다.'}
```

**코드 이해하기**

`ExportRequest`에는 `task`와 `limit`를 넣습니다. `task=null`은 두 교과의 기록을 모두 내보내고, `inspection`과 `action`을 선택하면 해당 기록만 내보냅니다. `limit`는 1~1000으로 제한합니다.

`pipeline_health()`는 실제 PostgreSQL과 Redis에 접근해 연결 여부를 확인하지만, **Worker가 살아 있는지까지 판정하지는 않습니다.** Worker 상태는 작업 등록 후 처리 로그와 최종 완료 여부로 별도로 확인합니다.

`job_status()`에서 Celery의 `FAILURE` 상태라면 결과 정보를 화면에 자세히 공개하지 않고, Worker 터미널 로그에서 실패 원인을 확인합니다. 오늘은 교육용으로 인증·권한을 별도 구현하지 않았으므로 로컬 또는 기관에서 허용한 개발망에서만 API를 사용합니다.

### 실습 16. 2일차 `app/main.py`에 Pipeline Router를 연결합니다

**1·2일차 코드를 삭제하지 않습니다.** `app/main.py`의 맨 아래에 다음 두 줄을 추가합니다.

**파일: `app/main.py` — 맨 아래에 추가할 부분**

```python
# 3일차 AI Native Data + Redis/Celery API를 기존 FastAPI에 연결합니다.
from app.pipeline_api import router as pipeline_router
app.include_router(pipeline_router)
```

2일차에 넣었던 다음 두 줄은 **반드시 그대로 둡니다.**

```python
from app.db_api import router as db_router
app.include_router(db_router)
```

이렇게 하면 FastAPI를 새로 만들지 않고 **하나의 앱에서 1일차 JSON API, 2일차 DB API, 3일차 비동기 작업 API**를 모두 사용할 수 있습니다.

### 실습 17. 비동기 Export 웹 화면을 작성합니다

이제 브라우저에서 내보내기를 요청하고 상태를 확인하는 화면을 만듭니다. 이전의 `web/index.html`과 `web/db.html`은 그대로 둡니다.

**의사코드**

```text
/pipeline 웹 페이지 열기
       ↓
PostgreSQL·Redis 준비 상태 확인 버튼 클릭
       ↓
Export 종류와 최대 건수 선택
       ↓
비동기 Export 요청 버튼 클릭
       ↓
FastAPI가 돌려준 작업 ID 화면에 출력
       ↓
상태 새로고침 버튼 클릭
       ↓
PENDING / STARTED / SUCCESS 확인
       ↓
SUCCESS이면 파일 상대경로와 건수 표시
```

**파일: `web/pipeline.html`**

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>교과 13 · 3일차 AI 데이터 Export</title>
  <style>
    body {font-family: system-ui, sans-serif; max-width: 920px; margin: 30px auto; padding: 0 16px; color: #243247;}
    section {border: 1px solid #d4dbe6; border-radius: 10px; margin: 14px 0; padding: 16px;}
    select, input, button {font: inherit; padding: 8px; margin: 4px;}
    button {cursor: pointer;}
    pre {background: #f4f6f9; padding: 14px; white-space: pre-wrap; overflow-wrap: anywhere;}
    small {color: #596779;}
  </style>
</head>
<body>
  <h1>교과 13 · AI Native 데이터 관리</h1>
  <p>3일차 — PostgreSQL에 이미 저장된 AI 결과를 JSONL, Parquet 및 집계 파일로 내보냅니다.</p>
  <p><small>이 기능은 새 AI 추론이 아닙니다. 작업은 Redis Queue를 거쳐 Celery Worker가 처리합니다.</small></p>
  <p><a href="/db">2일차 DB 조회</a> · <a href="/docs">API 문서</a></p>
  <section>
    <h2>환경 상태</h2>
    <button id="health">PostgreSQL / Redis 확인</button>
    <pre id="healthResult">확인 전</pre>
  </section>
  <section>
    <h2>백그라운드 Export 등록</h2>
    <label for="task">종류</label>
    <select id="task"><option value="">전체</option><option value="inspection">교과 9 OCR</option><option value="action">교과 10 Action</option></select>
    <label for="limit">최대 건수</label>
    <input id="limit" type="number" min="1" max="1000" value="100">
    <button id="run">비동기 Export 요청</button>
    <p id="jobText">아직 실행한 작업이 없습니다.</p>
    <button id="refresh" disabled>작업 상태 새로고침</button>
    <pre id="result">{}</pre>
  </section>
  <script>
    let currentJob = null;
    const resultView = document.getElementById('result');
    const jobText = document.getElementById('jobText');
    async function readJson(url, options) {
      const response = await fetch(url, options);
      const payload = await response.json();
      if (!response.ok) throw new Error(payload.detail ?? `HTTP ${response.status}`);
      return payload;
    }
    document.getElementById('health').addEventListener('click', async () => {
      const target = document.getElementById('healthResult');
      try {target.textContent = JSON.stringify(await readJson('/api/v1/pipeline/health'), null, 2);}
      catch (error) {target.textContent = error.message;}
    });
    document.getElementById('run').addEventListener('click', async () => {
      try {
        const choice = document.getElementById('task').value;
        const number = Number(document.getElementById('limit').value);
        const payload = await readJson('/api/v1/pipeline/exports', {
          method: 'POST', headers: {'Content-Type': 'application/json'},
          body: JSON.stringify({task: choice || null, limit: number})
        });
        currentJob = payload.job_id;
        jobText.textContent = `등록된 작업 ID: ${currentJob}`;
        document.getElementById('refresh').disabled = false;
        resultView.textContent = JSON.stringify(payload, null, 2);
      } catch (error) {resultView.textContent = `요청 실패: ${error.message}`;}
    });
    document.getElementById('refresh').addEventListener('click', async () => {
      if (!currentJob) return;
      try {
        const payload = await readJson(`/api/v1/pipeline/jobs/${encodeURIComponent(currentJob)}`);
        jobText.textContent = `작업 상태: ${payload.status}`;
        resultView.textContent = JSON.stringify(payload, null, 2);
      } catch (error) {resultView.textContent = `조회 실패: ${error.message}`;}
    });
  </script>
</body>
</html>
```

**웹 코드 이해하기**

`fetch()`는 앞에서 사용한 것과 동일하게 FastAPI에 요청을 보냅니다. 이번에는 `POST /api/v1/pipeline/exports`로 작업을 등록하고, `GET /api/v1/pipeline/jobs/{job_id}`로 상태를 확인한다는 점이 다릅니다.

상태가 바뀌는지 확인하는 버튼은 **사용자가 직접 누르는 새로고침 방식**입니다. 작업을 여러 번 등록하는 것과 상태를 여러 번 조회하는 것은 서로 다르므로, 조회할 때는 `비동기 Export 요청`을 다시 누르지 말고 `작업 상태 새로고침`을 사용합니다.

2일차 DB 조회 화면에서도 이동할 수 있도록 **기존 `web/db.html`의 `</body>` 바로 앞**에 다음 한 줄을 추가합니다.

```html
<p><a href="/pipeline">3일차 · Redis/Celery 데이터 내보내기</a></p>
```

### 실습 18. FastAPI 서버를 실행하고 실제 비동기 작업을 확인합니다

**현재 실행 중이어야 하는 구성**

```text
PostgreSQL 서버         실행 중
Redis 서버              실행 중
Celery Worker 터미널     실행 중
FastAPI 서버 터미널      지금 실행
```

Celery Worker 터미널은 그대로 둔 상태에서 **다른 새 WSL 터미널**을 열고 다음을 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
```

브라우저에서 다음 세 주소를 확인합니다.

```text
http://127.0.0.1:8000/docs
http://127.0.0.1:8000/api/v1/pipeline/health
http://127.0.0.1:8000/pipeline
```

`/api/v1/pipeline/health`의 연결이 정상이면 다음과 비슷합니다.

```json
{
  "status": "ok",
  "postgresql": true,
  "redis": true,
  "worker": "이 API는 Worker 실행 여부까지 확인하지 않습니다."
}
```

웹 화면에서 **전체 / 100건**을 선택한 뒤 **비동기 Export 요청**을 누릅니다. 화면에 작업 ID가 출력되면 작업 등록에 성공한 것입니다.

이어서 **작업 상태 새로고침**을 눌러 봅니다. 상황에 따라 `PENDING`, `STARTED`, `SUCCESS` 가운데 하나가 표시될 수 있습니다. Worker가 정상 처리한 후에는 `SUCCESS`가 나타나고 `result`에 다음 정보가 포함됩니다.

```json
{
  "job_id": "a3c704f0-a9fb-4b24-8fc7-cf7e8f20dc91",
  "status": "SUCCESS",
  "result": {
    "job_id": "a3c704f0-a9fb-4b24-8fc7-cf7e8f20dc91",
    "row_count": 11,
    "task_filter": null,
    "limit": 100,
    "schema_version": "day03-v1",
    "artifacts": {
      "bronze": "data/day03/exports/a3c704f0-a9fb-4b24-8fc7-cf7e8f20dc91/bronze_events.jsonl",
      "silver": "data/day03/exports/a3c704f0-a9fb-4b24-8fc7-cf7e8f20dc91/silver_events.parquet",
      "gold": "data/day03/exports/a3c704f0-a9fb-4b24-8fc7-cf7e8f20dc91/gold_summary.json",
      "manifest": "data/day03/exports/a3c704f0-a9fb-4b24-8fc7-cf7e8f20dc91/manifest.json"
    }
  }
}
```

실제 응답의 `result`에는 `completed_at_utc`도 포함되며 ID·건수·저장 시각은 달라집니다. `SUCCESS`를 확인한 뒤 터미널에서 다음을 실행합니다.

```bash
python -m scripts.06_inspect_export
```

**최종 확인:** Celery 작업을 실행해 생성된 결과도 Bronze·Silver·Gold 행 수가 같으면 비동기 내보내기가 완료된 것입니다. 2일차 DB의 기존 이벤트는 수정하거나 삭제하지 않았습니다.

---

## PART 6. 자동 테스트와 오류 확인

### 실습 19. 3일차 자동 테스트를 작성합니다

오늘 작업은 PostgreSQL 데이터를 기반으로 파일을 생성합니다. 그러나 자동 테스트를 실행할 때마다 실제 학생의 PostgreSQL에 새로운 실습 기록을 쌓거나 기존 기록을 변경해서는 안 됩니다.

**따라서 2일차와 같은 방식으로 테스트 전용 SQLite DB를 사용**합니다. 시험용으로 1일차의 공통 이벤트를 DB에 넣은 뒤 내보내기 결과를 확인합니다. Celery의 작업 등록 API는 작업 요청 함수를 테스트용으로 바꾸어 검사하므로, 테스트마다 실제 Redis Queue에 작업을 누적하지 않습니다.

**검사할 내용**

```text
테스트 전용 DB 생성
       ↓
1일차 공통 이벤트를 2일차 ORM 구조로 저장
       ↓
Bronze JSONL · Silver Parquet · Gold JSON 생성
       ↓
총 건수·NULL Confidence·UNKNOWN 유지 확인
       ↓
교과 필터와 수동 CRUD 결과 제외 확인
       ↓
파일 저장 전후 DB 이벤트 수가 변하지 않았는지 확인
       ↓
FastAPI 202 작업 등록과 상태 조회 응답 검증
       ↓
기존 1·2일차 테스트도 다시 실행
```

**파일: `tests/test_day03.py`**

```python
"""3일차: 기존 DB 기록을 변경하지 않고 Export 형식과 API를 검사합니다."""
import json
from uuid import uuid4

import pytest
import pyarrow.parquet as pq
from fastapi.testclient import TestClient
from sqlalchemy import create_engine, select, func
from sqlalchemy.orm import sessionmaker

from app.db import Base, get_db
from app.main import app
from app.models import InferenceEventRow
from app.records import load_events
from app.repository import save_recorded_events
from app import data_pipeline
from app import pipeline_api


@pytest.fixture
def context(tmp_path, monkeypatch):
    engine = create_engine(f'sqlite+pysqlite:///{tmp_path / "day03_test.sqlite"}')
    Base.metadata.create_all(engine)
    factory = sessionmaker(bind=engine, expire_on_commit=False)
    monkeypatch.setattr(data_pipeline, 'ROOT', tmp_path)
    monkeypatch.setattr(data_pipeline, 'EXPORT_ROOT', tmp_path / 'data' / 'day03' / 'exports')

    def override_db():
        with factory() as session:
            yield session

    app.dependency_overrides[get_db] = override_db
    try:
        with TestClient(app) as client:
            yield client, factory, tmp_path
    finally:
        app.dependency_overrides.clear()
        engine.dispose()


def seed(factory):
    with factory() as session:
        return save_recorded_events(session, load_events())


def test_bronze_silver_gold_keep_real_results(context):
    client, factory, tmp_path = context
    seed(factory)
    with factory() as db:
        before = db.scalar(select(func.count()).select_from(InferenceEventRow))
        manifest = data_pipeline.export_event_snapshot(db, str(uuid4()), limit=1000)
        after = db.scalar(select(func.count()).select_from(InferenceEventRow))
    assert before == after == len(load_events())
    root = tmp_path / 'data' / 'day03' / 'exports' / manifest['job_id']
    bronze = [json.loads(line) for line in (root / 'bronze_events.jsonl').read_text(encoding='utf-8').splitlines()]
    silver = pq.read_table(root / 'silver_events.parquet').to_pylist()
    gold = json.loads((root / 'gold_summary.json').read_text(encoding='utf-8'))
    assert manifest['row_count'] == len(bronze) == len(silver) == gold['total']
    assert all(item['asset_metadata']['asset_file_verified'] is False for item in bronze)
    assert all(item['event']['source_mode'] == 'recorded_result' for item in bronze)
    assert any(item['task'] == 'inspection' and item['confidence'] is None for item in silver)
    assert gold['by_task']['inspection'] + gold['by_task']['action'] == gold['total']
    assert sum(gold['by_decision'].values()) == gold['total']


def test_filter_is_separate_and_keeps_unknown(context):
    client, factory, tmp_path = context
    seed(factory)
    with factory() as db:
        result = data_pipeline.export_event_snapshot(db, str(uuid4()), task_filter='action', limit=1000)
    rows = pq.read_table(tmp_path / 'data' / 'day03' / 'exports' / result['job_id'] / 'silver_events.parquet').to_pylist()
    expected = [e for e in load_events() if e.task == 'action']
    assert len(rows) == len(expected)
    assert all(row['task'] == 'action' for row in rows)
    assert any(row['decision'] == 'UNKNOWN' for row in rows)


def test_manual_training_rows_not_misrepresented_as_ai(context):
    client, factory, tmp_path = context
    seed(factory)
    body = {'task':'inspection','input_ref':'manual.png','predicted_label':'OK',
            'expected_label':'OK','decision':'OK','reasons':[]}
    assert client.post('/api/v1/db/events', json=body).status_code == 201
    with factory() as db:
        result = data_pipeline.export_event_snapshot(db, str(uuid4()), limit=1000)
    assert result['row_count'] == len(load_events())


def test_empty_or_invalid_export_is_rejected(context):
    client, factory, tmp_path = context
    with factory() as db:
        with pytest.raises(ValueError, match='기존 AI 결과'):
            data_pipeline.export_event_snapshot(db, str(uuid4()))
        with pytest.raises(ValueError, match='UUID'):
            data_pipeline.export_event_snapshot(db, '../unexpected')


def test_enqueue_request_is_not_synchronous(context, monkeypatch):
    client, factory, tmp_path = context
    fake_id = str(uuid4())
    calls = []
    def fake_delay(**kwargs):
        calls.append(kwargs)
        return type('FakeJob', (), {'id':fake_id})()
    monkeypatch.setattr(pipeline_api.export_events_task, 'delay', fake_delay)
    response = client.post('/api/v1/pipeline/exports', json={'task':'inspection', 'limit':25})
    assert response.status_code == 202
    assert response.json()['job_id'] == fake_id
    assert calls == [{'task_filter':'inspection','limit':25}]
    assert client.post('/api/v1/pipeline/exports', json={'task':'bad', 'limit':25}).status_code == 422


def test_job_status_success_manifest(context, monkeypatch):
    client, factory, tmp_path = context
    job_id = str(uuid4())
    expected = {'job_id':job_id, 'row_count':3, 'artifacts': {'gold':'data/day03/exports/example/gold_summary.json'}}
    def fake_async_result(value, app):
        return type('FakeResult', (), {'state':'SUCCESS', 'result':expected})()
    monkeypatch.setattr(pipeline_api, 'AsyncResult', fake_async_result)
    response = client.get('/api/v1/pipeline/jobs/' + job_id)
    assert response.status_code == 200
    assert response.json()['result'] == expected


def test_job_status_id_must_be_uuid(context):
    client, factory, tmp_path = context
    assert client.get('/api/v1/pipeline/jobs/not-a-uuid').status_code == 422
```

프로젝트 최상위에서 실행합니다.

```bash
python -m pytest -q
```

**1일차 12개 + 2일차 7개 + 3일차 7개**를 모두 작성한 경우 **총 26개의 테스트**가 실행됩니다. 패키지가 제대로 설치되고 이전 결과 JSON이 준비되어 있다면 모든 테스트가 통과해야 합니다.

```text
..........................                                              [100%]
26 passed
```

이 테스트는 **데이터 변환, SQLAlchemy, FastAPI의 연결 로직**을 검사합니다. Redis 서버가 실제로 실행되는지, Celery Worker가 작업을 완료하는지, 강의실 PostgreSQL 접속이 올바른지는 **실습 8·14·18에서 실제 환경으로 별도 확인해야 합니다.** 특히 오늘의 Parquet 파일은 실습 10·11에서 실제 PyArrow로 저장하고 다시 읽어 확인합니다.

### 실습 20. 오류가 발생하면 발생 단계부터 확인합니다

| 증상 | 확인할 원인 | 해결 방법 |
|---|---|---|
| `No module named celery` / `redis` / `pyarrow` | `.venv`의 패키지 누락 | `source .venv/bin/activate` 후 `python -m pip install -r requirements.txt` |
| `redis-cli: command not found` | Redis CLI 미설치 | `sudo apt install redis-server redis-tools` |
| `Connection refused` (Redis) | Redis 서버 미실행 | `sudo service redis-server start`, `redis-cli ping` 확인 |
| `PostgreSQL 연결 실패` | 2일차 DB 서버·`.env` 오류 | `pg_isready`, `.env`, DB 사용자·비밀번호 확인 |
| 내보낼 기존 AI 기록이 없다는 오류 | `recorded_result` 0건 | `python -m scripts.03_import_records_to_db` 실행 |
| Celery Worker에 `day03.export_events`가 보이지 않음 | 작업 모듈을 읽지 못함 | `celery_app.py`의 `include`, Worker 실행 명령 확인 |
| 작업 상태가 계속 `PENDING` | Worker 미실행·Queue 문제·만료 또는 잘못된 ID | Worker 터미널 로그, 작업 ID, Redis 연결 확인 |
| 상태 `FAILURE` | Worker 작업 중 예외 | Worker 로그와 동기 내보내기 `05_export_once.py` 실행 결과 확인 |
| `ImportError` / Arrow 관련 오류 | PyArrow·Python 버전/환경 불일치 | 현재 `.venv`에서 지원되는 PyArrow 버전 재확인 |
| `manifest.json`이 생성되지 않음 | 파일 저장 중 오류로 작업 미완료 | Worker 로그 확인, 해당 작업이 SUCCESS인지 확인 |
| 기존 API `/db`가 사라짐 | `app/main.py`에서 2일차 Router 삭제 | 2일차 Router 코드를 복구한 뒤 3일차 Router 추가 |
| `.env` 또는 민감 자료가 Git에 표시됨 | `.gitignore` 규칙 누락 또는 기존 추적 | `git status`·`git check-ignore` 확인, 공개 업로드 중지 |

**문제 해결 순서:** 2일차 DB 기록 → 동기 Export → 실제 Parquet 검증 → Redis `PONG` → Celery Worker → FastAPI 작업 등록 → 작업 상태 확인 → 웹 화면 순서로 점검합니다. **비동기 요청이 실패했다고 DB 테이블이나 이전 교과 JSON을 삭제하지 않습니다.**

---

## PART 7. 데이터 구조 문서·README·Git 정리

### 실습 21. 3일차 데이터 설계 문서를 작성합니다

오늘 만든 파일 이름과 각각의 역할, 다음 일차에서 연결할 위치를 문서로 남깁니다. 이 문서를 읽으면 나중에 데이터 처리 코드를 다시 열지 않아도 어떤 데이터가 어디에 저장되어 있는지 알 수 있습니다.

**파일: `docs/day03_data_design.md`**

````markdown
# 교과 13 · 3일차 AI Native 데이터와 비동기 구조

## 입력

- 2일차 PostgreSQL `inference_events`
- `source_mode=recorded_result`만 Export
- 원본 이미지·영상 자체는 복사하지 않음
- `input_ref`, `asset_kind`는 파일 참조 메타데이터

## Bronze / Silver / Gold

| 계층 | 파일 | 역할 |
|---|---|---|
| Bronze | `bronze_events.jsonl` | 원본 공통 이벤트 스냅샷 + 입력 참조 정보 |
| Silver | `silver_events.parquet` | 자료형을 통일한 분석용 열 데이터 |
| Gold | `gold_summary.json` | 교과별·판정별 저장 기록 수 |

- 작업별 출력 폴더: `data/day03/exports/<작업_UUID>/`
- 완료 기준: `manifest.json` 생성 및 Parquet 행 수 검증
- OCR의 `confidence=null` 유지
- Action의 `UNKNOWN`을 NG로 변경하지 않음
- `recorded_at_raw` 원본 시각 유지

## 비동기 서비스

브라우저 → FastAPI `/api/v1/pipeline/exports`
→ Redis Broker → Celery Worker
→ PostgreSQL 조회 → Bronze/Silver/Gold 저장
→ Redis Result Backend → FastAPI 작업 상태 조회

- Worker 실행: `celery -A app.celery_app:celery_app worker --loglevel=info --pool=solo`
- 저장 작업: `day03.export_events`
- 조회 API: `/api/v1/pipeline/jobs/{job_id}`
- Redis 작업 결과는 일정 시간 후 만료될 수 있으나 파일은 남음

## 연결 범위

- 1일차: 공통 JSON
- 2일차: PostgreSQL ORM·CRUD
- 3일차: 구조화 데이터 Export + 비동기 Worker
- 4일차: 실제 카메라/모델 추론 결과를 PostgreSQL·플랫폼에 연결
- 7일차: Docker·Compose로 서비스 실행 환경 구성

## 보안

`.env`, 원본 데이터, Export 결과는 공개 Git에 포함하지 않음.
````

### 실습 22. 기존 README와 시스템 구조도를 업데이트합니다

1일차부터 계속 업데이트한 `docs/architecture.md`에 다음 내용을 **맨 아래에 추가**합니다. 기존 설명을 삭제하지 않습니다.

**파일: `docs/architecture.md` — 추가할 내용**

```markdown
## 3일차 완성: AI Native 데이터 Export와 비동기 Worker

PostgreSQL 저장 결과 → data_pipeline.py → Bronze JSONL / Silver Parquet / Gold JSON

브라우저 → FastAPI → Redis Broker → Celery Worker
                                   ↓
                          PostgreSQL 이벤트 조회
                                   ↓
                          JSONL·Parquet·Gold 파일
                                   ↓
                             작업 상태 조회

- 실제 카메라/새 추론 결과 연결은 4일차
- Docker·Compose 통합은 7일차
```

또한 `README.md`에도 아래 내용을 **추가**합니다. 1·2일차 실행 명령을 지우지 않습니다.

**파일: `README.md` — 추가할 내용**

````markdown
## 3일차 — Redis·Celery와 AI Native 데이터

### 사전 준비

- PostgreSQL 실행 및 2일차 `recorded_result` 데이터 이관 완료
- Redis 실행 (`redis-cli ping` → `PONG`)
- `.env`에 `REDIS_URL=redis://127.0.0.1:6379/0` 설정
- `python -m pip install -r requirements.txt`

### 실행 순서

**터미널 A — Redis 연결 점검**
```bash
redis-cli ping
python -m scripts.04_check_day03_inputs
```

**터미널 B — Celery Worker**
```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
celery -A app.celery_app:celery_app worker --loglevel=info --pool=solo
```

**터미널 C — FastAPI**
```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

**터미널 D — 파일 결과 및 테스트 확인**
```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m scripts.06_inspect_export
python -m pytest -q
```

- 화면: http://127.0.0.1:8000/pipeline
- DB 조회: http://127.0.0.1:8000/db
- API 문서: http://127.0.0.1:8000/docs
- 생성 데이터: `data/day03/exports/`

### 오늘의 기능 범위

2일차 PostgreSQL에 저장된 **기존 AI 결과**를 내보냅니다.
실제 카메라·YOLO·OCR 추론과 대용량 영상 저장은 아직 실행하지 않습니다.
````

### 실습 23. `.gitignore`에 Export 결과 폴더를 추가합니다

1·2일차에 `.venv`, `.env`, `handoff/records`, `handoff/provided`는 이미 Git에서 제외했습니다. **기존 규칙은 그대로 유지**하고 `.gitignore` 맨 아래에 다음 한 줄을 추가합니다.

```gitignore
data/day03/exports/
```

이 폴더에는 이전 AI 결과에서 생성한 데이터가 들어 있으므로 수업용 공개 저장소에 기본으로 올리지 않습니다.

다음 명령으로 무시 규칙이 적용되는지 확인합니다.

```bash
git check-ignore -v data/day03/exports/*/manifest.json
```

작업 폴더가 아직 없으면 아무 내용도 출력되지 않을 수 있습니다. 오늘 앞선 내보내기를 완료했다면 해당 파일과 `.gitignore`의 규칙이 나타납니다.

---

# 6. 오늘 작성한 내용의 최종 정리 및 GitHub 업로드

## 6.1. 오늘의 최종 결과

1일차에는 교과 9·10 결과를 하나의 공통 JSON으로 변환했습니다. 2일차에는 그 결과를 PostgreSQL에 저장하고 CRUD를 작성했습니다. 오늘 3일차에는 PostgreSQL의 **기존 AI 기록**에서 다음 세 가지 데이터 결과를 만들었습니다.

```text
1일차 InferenceEvent (공통 JSON)
              ↓
2일차 PostgreSQL 저장 + CRUD
              ↓
3일차 DB 기록 스냅샷 내보내기
              ↓
      ├── Bronze JSONL
      ├── Silver Parquet
      └── Gold 요약 JSON
              ↑
      Redis + Celery Worker
              ↑
      FastAPI 비동기 요청
```

**이 구조의 핵심**은 AI 분석 결과를 저장·관리하기 쉬운 자료형으로 바꾸고, 파일 생성처럼 시간이 걸릴 수 있는 작업을 웹 요청과 분리했다는 것입니다. 아직 새 이미지·영상에 대한 AI 추론은 수행하지 않았습니다.

## 6.2. 최종 자가 체크리스트

- [ ] 2일차 `subject13_ai_platform`과 `.venv`를 그대로 사용했습니다.
- [ ] PostgreSQL의 `recorded_result` 기록을 확인했습니다.
- [ ] Redis에서 `PONG`을 확인했습니다.
- [ ] `04_check_day03_inputs.py`로 PostgreSQL·Redis 준비 상태를 확인했습니다.
- [ ] 동기 `05_export_once.py` 실행 후 네 가지 파일이 생성되었습니다.
- [ ] Bronze JSONL과 Silver Parquet의 데이터 구조를 확인했습니다.
- [ ] Gold 요약에서 `OK/NG/UNKNOWN`이 구분되며 건수가 맞습니다.
- [ ] 원래 Confidence가 없는 OCR의 값이 `null`로 남아 있습니다.
- [ ] `manual_entry`는 기존 AI 결과로 오인되어 Export되지 않았습니다.
- [ ] Celery Worker에 `day03.export_events`가 등록되어 있습니다.
- [ ] `/pipeline`에서 작업 ID를 받고 `SUCCESS`와 산출물 목록을 확인했습니다.
- [ ] `python -m scripts.06_inspect_export`의 최종 결과가 `PASS`입니다.
- [ ] 1·2일차 테스트와 새 3일차 테스트가 모두 통과했습니다.
- [ ] README·시스템 구조도·데이터 설계 문서를 업데이트했습니다.
- [ ] `.env`, 기존 교과 기록, Parquet·JSONL 파일은 Git 업로드 대상에 없습니다.

## 6.3. 3일차 Git 커밋을 남깁니다

프로젝트 최상위에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git status
```

오늘 추가한 소스와 문서만 스테이징합니다.

```bash
git add requirements.txt .env.example .gitignore \
  app/main.py app/data_pipeline.py app/celery_app.py app/worker_tasks.py app/pipeline_api.py \
  scripts/04_check_day03_inputs.py scripts/05_export_once.py scripts/06_inspect_export.py \
  web/pipeline.html web/db.html tests/test_day03.py \
  docs/day03_data_design.md docs/architecture.md README.md

git status
```

`git status`에서 `data/day03/exports/`, `.env`, `handoff/records/`, `handoff/provided/`가 스테이징되지 않았는지 확인합니다. 그다음 커밋합니다.

```bash
git commit -m "교과13 3일차: AI Native JSONL Parquet와 Redis Celery 비동기 처리"
git log -1 --oneline
git status
```

**2일차까지 사용하던 동일한 Git 저장소**에 3일차 작업이 기록됩니다. 지금까지의 커밋과 기록을 삭제하지 않습니다.

## 6.4. GitHub 또는 허용된 내부 저장소에 업로드합니다

이전에 원격 저장소를 연결해 두었고 외부 업로드가 허용되는 교육환경이라면 다음 명령을 실행합니다.

```bash
git remote -v
git push origin main
```

GitHub 사용이 제한된 교육장에서는 로컬 Git 커밋까지만 진행하거나 승인된 내부 GitLab/Gitea에 업로드합니다. **학습·추론 모델 가중치, 개인정보가 포함될 수 있는 데이터, `.env` 비밀번호, 원본 검사 파일, Export 데이터는 허가 없이 공개 저장소에 올리지 않습니다.**

## 6.5. 4일차 수업과 어떻게 연결되나요?

오늘까지는 이미 저장된 교과 9·10 결과를 웹·PostgreSQL·데이터 파일로 관리하는 방법을 완성했습니다.

**4일차에는 교과 9의 OCR 검사 함수와 교과 10의 Pose·Action Recognition 실행 코드를 실제 영상 입력에 연결**합니다. 새로운 추론 결과도 1일차의 공통 이벤트 규격을 따르도록 만들고, 2일차의 PostgreSQL에 저장할 수 있게 확장합니다. 오늘 작성한 Redis·Celery 작업은 결과 후처리와 데이터 내보내기에 그대로 활용할 수 있습니다.

```text
1~3일차
기존 AI 결과 → 공통 JSON → PostgreSQL → JSONL/Parquet/Gold → 웹 조회

4일차
실시간 영상 → 실제 AI 추론 → 공통 JSON → PostgreSQL → 웹 대시보드
                                                ↓
                                       Redis·Celery 후처리 연결
```

**다음 날도 오늘 만든 `subject13_ai_platform` 폴더와 가상환경을 그대로 사용합니다.**

