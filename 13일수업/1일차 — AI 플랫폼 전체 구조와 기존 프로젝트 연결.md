## 1. 오늘의 목표

교과 9에서는 OCR로 제품의 문자 정보를 검사하고, FastAPI로 검사 결과를 웹 화면에 전달했습니다. 교과 10에서는 Pose와 Action Recognition 결과를 JSON과 웹 화면으로 확인했습니다.

오늘은 **교과 9와 10에서 직접 만든 결과를 하나의 공통 AI 플랫폼으로 연결**합니다. 서로 다른 결과를 공통 구조로 정리하고, FastAPI와 웹브라우저에서 함께 조회하는 프로그램을 직접 작성합니다.

오늘 수업이 끝나면 다음 작업을 수행할 수 있습니다.

- 앞서 만든 교과 9·10 프로젝트에서 필요한 결과 JSON을 찾을 수 있습니다.
- 새로운 `subject13_ai_platform` 폴더와 Python 가상환경을 만들 수 있습니다.
- 교과 9·10의 서로 다른 JSON을 공통 `InferenceEvent` 구조로 변환할 수 있습니다.
- FastAPI에서 기록 목록, 상세 기록, 요약 정보를 제공할 수 있습니다.
- 웹 화면에서 제조 검사와 동작인식 결과를 선택하여 조회할 수 있습니다.
- 자동 테스트를 실행하고 README·Git에 오늘 작업을 기록할 수 있습니다.

**오늘 완성할 결과:** 교과 9의 OCR 검사 기록과 교과 10의 Action Recognition 기록을 한 웹 화면에서 조회하는 공통 플랫폼입니다. 이전 수업에서 생성한 기록의 개수는 학생마다 다를 수 있습니다.

---

## 2. 오늘 수업의 전체 흐름

![[Pasted image 20261018125230.png]]

```text
교과 9 결과                         교과 10 결과
OCR 검사 이력                        Action Recognition 결과
service_history.json               jobs/*/result.json
        │                                  │
        └────────────────┬─────────────────┘
                         ↓
                이전 결과 파일 확인
                         ↓
                교과 13 폴더 생성
                         ↓
              새 Python .venv 준비
                         ↓
            원본 JSON을 별도 폴더에 보존
                         ↓
           공통 InferenceEvent 구조 만들기
                         ↓
           교과 9·10 JSON 변환 코드 작성
                         ↓
              FastAPI 기록 조회 API
                         ↓
              HTML 웹 조회 화면 연결
                         ↓
                  결과·테스트 확인
                         ↓
                  README · Git 정리
                         ↓
         2일차 PostgreSQL·SQLAlchemy 연결
```

오늘은 이전 수업에서 완성한 OCR과 Action Recognition 알고리즘을 다시 작성하지 않습니다. 
**기존에 저장한 실제 검사 결과를 읽어 공통 플랫폼에 연결**하는 것이 오늘의 실습입니다.

새 카메라로 영상을 촬영하거나 AI 모델을 다시 실행하는 기능은 **4일차 실시간 AI 서비스**에서 
추가합니다. 오늘 사용한 데이터는 실제 이전 수업의 산출물이지만, 현재 화면에 표시되는 것은 
**저장된 결과의 조회**입니다.

---

## 3. 실습 전에 알아야 할 핵심

### 3.1. 교과 9·10에서 무엇을 가져오나요?

교과 9의 10일차에서는 OCR 검사 기능을 FastAPI 웹 화면에 연결했습니다. 제품을 검사하면 OK/NG 결과가 화면에 표시되고, 검사 기록은 `service_history.json` 파일에 저장됩니다.

교과 10의 10일차에서는 Action Recognition 실행 결과를 `result.json` 파일로 기록하도록 구성했습니다.

**오늘 사용할 파일**

|이전 수업|필요한 파일|역할|
|---|---|---|
|교과 9 · 10일차|`reports/day10/service_history.json`|OCR 검사 이력|
|교과 10 · 10일차|`reports/day10/web/jobs/*/result.json`|동작인식 결과|

오늘은 기존 AI 모델을 다시 실행하지 않고, **두 교과에서 저장한 JSON 결과를 하나의 공통 형식으로 정리하여 FastAPI 웹 화면에서 조회**합니다.

실제 OCR과 Pose·Action Recognition 추론은 4일차에 연결합니다.

### 3.2. 두 교과의 JSON 결과는 왜 다를까요?

교과 9와 교과 10에서는 FastAPI를 이용해 검사 결과를 웹 화면에 표시하고 JSON 파일로 저장했습니다.

하지만 검사 목적이 다르기 때문에 저장하는 항목도 다릅니다.

**① 교과 9 — OCR 검사 결과**

저장 파일: `reports/day10/service_history.json`

```json
{
  "file_name": "0679256889404_side.png",
  "final_decision": "NG",
  "reasons": ["NG_EXPECTED_MISMATCH"]
}
```

제품 이미지에서 읽은 문자와 기준 문자를 비교하고, **OK/NG 판정과 그 이유**를 저장합니다.

위 결과는 OCR이 읽은 문자열과 기준 문자열이 달라 NG가 된 경우입니다.

**② 교과 10 — Action Recognition 결과**

저장 파일: `reports/day10/web/jobs/작업ID/result.json`

```json
{
  "source_file": "S03_A006_T01.mp4",
  "expected_action": "wave",
  "detected_action": "wave",
  "latest_confidence": 0.2933,
  "final_match": true
}
```

영상에서 예측한 동작과 기대 동작을 비교하고, **예측 결과·신뢰도·일치 여부**를 저장합니다.

위 결과는 `wave` 동작을 예측했고 실제 기대 동작과 일치한 경우입니다.

두 JSON은 모두 실제 10일차 코드에 있는 결과이며, 여기서는 이해하기 쉽도록 주요 항목만 발췌했습니다.

**③ 왜 공통 JSON으로 바꿔야 할까요?**

|구분|교과 9|교과 10|
|---|---|---|
|검사 대상|제품 문자|사람의 동작|
|결과 필드|`final_decision`|`final_match`|
|신뢰도|저장하지 않음|`latest_confidence`|

교과 9에는 신뢰도 값이 없으므로 교과 13에서는 임의의 숫자를 만들지 않고 `null`(값 없음)로 처리합니다.

**오늘은 이렇게 서로 다른 두 JSON 결과를 공통** `**InferenceEvent**` **형식으로 변환하여 하나의 AI 플랫폼에서 조회하겠습니다.**

### 3.3. 공통 JSON 구조란 무엇인가요?

교과 9의 OCR 검사 결과와 교과 10의 동작인식 결과는 저장하는 항목의 이름이 다릅니다.

교과 13에서는 두 결과를 하나의 플랫폼에서 관리하기 위해 `**InferenceEvent**`**라는 공통 JSON 형식으로 변환**합니다.

기존 JSON 파일은 그대로 보존하고, 필요한 정보를 다음과 같이 같은 이름으로 정리합니다.

|공통 필드|교과 9 · OCR|교과 10 · 동작인식|
|---|---|---|
|`task`|`inspection`|`action`|
|`input_ref`|`file_name`|`source_file`|
|`recorded_at`|`inspected_at`|`created_at`|
|`predicted_label`|`ocr_text`|`detected_action`|
|`expected_label`|`expected_text`|`expected_action`|
|`confidence`|`null` (기록 없음)|`latest_confidence`|
|`decision`|`final_decision`|`final_match`를 기준으로 변환|
|`reasons`|기존 NG 판정 이유|동작 비교 결과에 따라 생성|

추가로 `event_id`(고유 ID), `source_path`(원본 파일 경로), `details`(교과별 상세 결과)와 출처·처리기 정보도 함께 기록합니다.

**공통 판정값은 다음 세 가지입니다.**

- `OK`: OCR 검사 통과 또는 기대 동작과 예측 동작이 일치
    
- `NG`: OCR 검사 실패 또는 확정된 예측 동작이 기대 동작과 불일치
    
- `UNKNOWN`: 동작을 예측하지 못했거나 최종 결과를 확정할 수 없음
    

동작인식의 `NG`가 곧 위험행동을 의미하지는 않습니다.

이렇게 통일한 결과는 이후 **PostgreSQL 데이터베이스에 저장하고, 웹 대시보드에서 통합 조회·관리**하는 데 사용합니다.

**즉, AI 모델을 합치는 것이 아니라 서로 다른 AI 검사 결과를 하나의 플랫폼에서 관리하기 위해 JSON 형식을 통일하는 것입니다.**

위의 내용을 쉽게 비유해서 다시 설명하면 
교과 9의 OCR 검사 결과와 교과 10의 동작인식 결과는 서로 다른 양식으로 작성된 보고서와 같으므로, 두 보고서를 하나의 공통 양식(`InferenceEvent`)으로 정리하여 나중에 데이터베이스에서 함께 저장하고 관리하기 위한 것입니다.

### 3.4. 오늘 가상환경과 Docker는 어떻게 사용하나요?

오늘은 새 교과 13 프로젝트에 **Python 가상환경(`.venv`)을 생성**합니다. 기존 교과 9·10의 `.venv`를 복사하지 않습니다.

교과 13의 **7일차에 Docker·Docker Compose**를 배우면서 지금까지 만든 서비스를 컨테이너로 실행합니다. 오늘은 Docker를 사용하지 않습니다.

---

## 1일차에 작성하는 주요 파일과 역할

이번 수업에서는 교과 9·10의 기존 검사 결과를 가져와 공통 JSON으로 변환하고, 
FastAPI와 웹 화면에서 함께 조회하는 프로그램을 만듭니다.

| 구분         | 파일명                            | 역할                                | 쉽게 이해하기                           |
| ---------- | ------------------------------ | --------------------------------- | --------------------------------- |
| 환경 설정      | `requirements.txt`             | 필요한 Python 패키지 목록                 | 프로그램 실행에 필요한 라이브러리 목록             |
| Python 설정  | `app/__init__.py`              | `app`을 Python 패키지로 구성             | `app` 폴더의 Python 파일을 불러올 수 있도록 준비 |
| 데이터 가져오기   | `scripts/01_import_results.py` | 교과 9·10의 기존 JSON 결과를 읽고 저장        | 두 교과의 검사 기록을 교과 13으로 가져오는 프로그램    |
| 공통 데이터 규격  | `app/schemas.py`               | `InferenceEvent` 데이터 구조와 검증 규칙 정의 | 두 교과가 사용할 공통 보고서 양식을 정하는 파일       |
| 데이터 변환     | `app/records.py`               | OCR·동작인식 결과를 공통 구조로 변환            | 서로 다른 보고서를 같은 양식으로 정리하는 파일        |
| API 서버     | `app/main.py`                  | FastAPI로 기록 목록·상세·통계 조회 기능 제공     | 정리된 검사 결과를 웹에 전달하는 프로그램           |
| 웹 화면       | `web/index.html`               | HTML·CSS·JavaScript로 조회 화면 구성     | 두 교과의 결과를 한 화면에서 보여주는 파일          |
| 자동 테스트     | `tests/test_platform.py`       | JSON 변환 및 API의 정상 동작 검사           | 프로그램이 올바르게 작동하는지 자동 확인            |
| 시스템 설명서    | `docs/architecture.md`         | 플랫폼 구성과 데이터 처리 흐름 기록              | 프로그램들이 어떻게 연결되는지 설명하는 문서          |
| 데이터 인수 설명서 | `handoff/README.md`            | 이전 교과 결과의 출처와 가져오는 방법 기록          | 어떤 데이터를 어디에서 가져왔는지 설명하는 문서        |
| 프로젝트 안내서   | `README.md`                    | 프로젝트 소개·설치·실행 방법 정리               | 다른 사람이 프로젝트를 실행할 수 있도록 안내         |
| Git 설정     | `.gitignore`                   | Git에서 제외할 파일과 폴더 지정               | 가상환경과 원본 데이터가 실수로 업로드되지 않도록 설정    |

### 실습 중 자동으로 생성되는 결과 파일

다음 두 파일은 직접 코드를 작성하는 파일이 아닙니다. `01_import_results.py`를 실행하면 기존 교과의 JSON 데이터를 읽어 자동으로 저장합니다.

|파일명|저장 내용|
|---|---|
|`handoff/records/inspection_history.json`|교과 9의 OCR 검사 기록|
|`handoff/records/action_jobs.json`|교과 10의 작업별 동작인식 결과 목록|

또한 이전 교과의 결과가 없는 학생은 강사가 제공한 자료를 `handoff/provided/` 폴더에 배치하여 사용합니다.

### 파일들이 연결되는 순서

```
01_import_results.py
    기존 교과 결과 가져오기
           ↓
schemas.py
    공통 데이터 규격 정의
           ↓
records.py
    공통 InferenceEvent로 변환
           ↓
main.py
    FastAPI로 조회 기능 제공
           ↓
index.html
    웹 화면에 결과 표시
           ↓
test_platform.py
    전체 기능 자동 검증
```

**핵심:** 오늘 작성하는 파일들은 각각 데이터 가져오기, 공통 형식 정의, 데이터 변환, API 제공, 웹 화면 표시, 자동 검증의 역할을 나누어 담당합니다. 이렇게 나누면 다음 수업에서 PostgreSQL과 새로운 AI 기능을 추가할 때도 기존 코드를 재사용하기 편리합니다.

---

## 4. 오늘 사용할 기술 스택

| 기술 | 오늘 사용하는 목적 |
|---|---|
| Python · `.venv` | 공통 플랫폼을 위한 독립 실행환경 구성 |
| `pathlib` · `json` | 이전 프로젝트의 실제 결과 파일 찾기·읽기 |
| Pydantic | 공통 JSON 필드와 자료형 검증 |
| FastAPI · Uvicorn | 공통 이벤트 조회 API 실행 |
| HTML · CSS · JavaScript `fetch()` | 웹에서 조회 목록·상세 결과 표시 |
| Pytest · HTTPX | API와 JSON 변환 결과 자동 검증 |
| Git | 하루 작업을 버전으로 기록 |

이번 실습은 **Windows 11 + WSL Ubuntu 또는 Ubuntu 환경**의 터미널 명령을 기준으로 합니다. VS Code, Python, Git이 준비되어 있어야 합니다. Raspberry Pi·Jetson·PostgreSQL은 오늘 필요하지 않습니다.

---

# 5. 처음부터 끝까지 따라 하는 상세 실습

## PART 1. 교과 13 프로젝트와 Python 환경 준비

### 실습 1. 교과 13 프로젝트 폴더를 만듭니다

교과 9·10의 기존 코드는 그대로 보존하고, 13일 동안 확장할 새 플랫폼 프로젝트를 만듭니다. WSL Ubuntu 터미널에서 입력합니다.

```bash
mkdir -p ~/ai_vision/subject13_ai_platform
cd ~/ai_vision/subject13_ai_platform
code -r .
mkdir -p app scripts web tests docs handoff/records handoff/provided/subject9 handoff/provided/subject10
pwd 
```

`pwd`의 마지막이 `subject13_ai_platform`이면 정상입니다. `code .`는 현재 폴더를 VS Code로 
여는 명령입니다. 이후 명령은 별도 안내가 없다면 **항상 프로젝트 최상위 폴더에서 실행**합니다.

최종적으로 다음과 같은 폴더와 파일이 만들어집니다. **지금은 폴더만 만들고 파일은 아래 실습에서 하나씩 작성합니다.**

```text
subject13_ai_platform/
├── .venv/                          # Python 가상환경
├── app/
│   ├── __init__.py
│   ├── schemas.py                  # 공통 JSON 데이터 규격
│   ├── records.py                  # 이전 교과 결과 변환
│   └── main.py                     # FastAPI 서버
├── scripts/
│   └── 01_import_results.py       # 기존 프로젝트 결과 인수
├── web/
│   └── index.html                  # 결과 조회 화면
├── tests/
│   └── test_platform.py           # 자동 검사
├── handoff/
│   ├── provided/                   # 미완성 시 제공받는 결과
│   │   ├── subject9/
│   │   └── subject10/
│   ├── records/                    # 인수한 결과 JSON
│   └── README.md
├── docs/
│   └── architecture.md
├── requirements.txt
├── .gitignore
└── README.md
```

### 실습 2. Python 가상환경을 생성합니다

교과 13은 교과 9·10과 별도의 프로젝트이므로 새 가상환경을 만듭니다. 앞 교과에서 사용하던 PyTorch·OCR 패키지는 오늘 설치하지 않아도 됩니다.

```bash
python3 --version
python3 -m venv .venv
source .venv/bin/activate
which python
python -m pip --version
```

`which python` 출력에 `subject13_ai_platform/.venv/bin/python`이 포함되어 있다면 올바른 환경입니다.

`ensurepip is not available` 오류가 나면 아래 명령으로 현재 Python 버전에 맞는 venv 패키지를 설치한 뒤, **현재 교과 13 폴더에서만** 가상환경을 다시 생성합니다.

```bash
sudo apt update
sudo apt install -y python3-venv
python3 -m venv .venv
source .venv/bin/activate
```

특정 Python 버전을 설치한 환경에서는 해당 버전의 `python3.x-venv` 패키지가 필요할 수 있습니다. 오류가 계속되면 `python3 --version`과 설치된 venv 패키지를 확인합니다.

### 실습 3. 이전 교과의 실제 결과 파일을 확인합니다

먼저 교과 9·10에서 사용한 기존 프로젝트를 확인합니다. 다음 경로는 교과 9 10에서 사용한 기본 경로입니다.

```bash
ls -lh ~/ai_vision/subject9_machinevision/reports/day10/service_history.json
find ~/ai_vision/subject10_pose_action/reports/day10/web/jobs   -type f -name "result.json" | sort
```

두 교과의 결과 파일이 보이면 별도로 복사하지 않아도 됩니다. 뒤에서 작성할 `01_import_results.py`가 **기존 위치를 직접 읽어** 교과 13에 필요한 데이터만 보관합니다.

**경로가 다르다면:** 파일이 없다고 바로 다시 만들지 말고 기존 프로젝트의 실제 저장 위치를 확인합니다. 다음 명령으로 파일을 찾을 수 있습니다.

```bash
find ~/ai_vision -type f -name "service_history.json"
find ~/ai_vision -type f -name "result.json" | grep '/day10/web/jobs/'
```

기존 프로젝트가 `~/ai_vision`이 아닌 위치에 있다면 탐색할 시작 폴더를 본인의 실제 경로로 바꾸세요. 찾은 경로는 **실습 6의 `SUBJECT9`, `SUBJECT10`**에 반영합니다.

#### 앞 교과 결과가 없는 경우

이전 수업을 완료하지 못했다면 1일차테이터 폴더안에 결과 파일을 아래와 같은 폴더 구조로 배치합니다. `result.json`은 같은 파일명이 여러 개이므로 **각 작업ID 폴더를 유지**해야 합니다.

```text
handoff/provided/
├── subject9/
│   └── reports/day10/
│       └── service_history.json
└── subject10/
    └── reports/day10/web/jobs/
        ├── job_7d93a2a2c0f6/
        │   └── result.json
        └── job_예시02/
            └── result.json
```

교과 9는 검사 결과가 1건 이상 포함된 JSON 배열, 교과 10은 작업별 `result.json` 파일이 1개 
이상 있으면 오늘 실습을 진행할 수 있습니다. 실제 결과 개수는 학생의 작업 내역과 제공 자료에 따라 달라집니다. **`handoff/provided/`는 Git에 업로드하지 않습니다.**

### 실습 4. 사용할 패키지를 설치합니다

VS Code에서 프로젝트 최상위에 `requirements.txt`를 생성하고 다음 내용을 저장합니다.

**파일: `requirements.txt`**

```text
fastapi>=0.115,<1
uvicorn[standard]>=0.30,<1
pydantic>=2,<3
httpx>=0.27,<1
pytest>=8,<10
```

FastAPI는 웹 API, Pydantic은 데이터 검증, Uvicorn은 서버 실행, Pytest와 HTTPX는 자동 테스트에 사용합니다.

이제 패키지를 설치합니다.

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m pip show fastapi uvicorn pydantic pytest httpx
```

설치 명령어 설명

|명령어|역할|
|---|---|
|`python -m pip install --upgrade pip`|Python 패키지를 설치하는 도구인 pip를 최신 버전으로 업데이트합니다.|
|`python -m pip install -r requirements.txt`|프로젝트에 필요한 패키지들을 `requirements.txt`에 적힌 목록대로 한 번에 설치합니다.|
|`python -m pip show fastapi uvicorn pydantic pytest httpx`|해당 패키지들이 설치되어 있는지 버전과 설치 위치를 확인합니다.|

### 주요 패키지 설명

- FastAPI: Python으로 웹 서버의 API를 만드는 프레임워크입니다.
    
- Uvicorn: FastAPI로 만든 웹 서버를 실제로 실행해 주는 프로그램입니다.
    
- Pydantic: API로 전달받은 데이터가 정해진 형식에 맞는지 검사합니다.
    
- Pytest: 작성한 Python 코드가 정상적으로 작동하는지 자동으로 테스트합니다.
    
- HTTPX: 다른 서버의 API에 HTTP 요청을 보내고 응답을 받는 도구입니다.
    

교과 13에서의 역할: FastAPI로 API를 만들고 → Uvicorn으로 실행하며 → Pydantic으로 데이터를 검증하고 → HTTPX와 Pytest로 API가 정상적으로 동작하는지 테스트합니다.

### 실습 5. Python 패키지 파일을 만듭니다

**파일: `app/__init__.py`**

```python
"""교과 13 공통 AI 플랫폼 앱 패키지."""
```

`app` 폴더의 Python 파일을 `app.main`, `app.records`처럼 불러오는 데 사용하는 파일입니다.

---

## PART 2. 교과 9·10의 저장 결과를 가져옵니다

### 실습 6. 이전 교과 결과를 읽는 코드를 작성합니다

이번 실습에서는 교과 9와 10에서 **이미 만들어 놓은 JSON 검사 결과를 교과 13 프로젝트로 가져옵니다.** AI 모델을 다시 실행하지는 않습니다.

두 교과는 결과를 저장한 방식이 다릅니다.

|구분|기존 결과 파일|저장된 내용|
|---|---|---|
|교과 9|`reports/day10/service_history.json`|하나의 JSON 파일에 여러 OCR 검사 기록 저장|
|교과 10|`reports/day10/web/jobs/작업ID/result.json`|작업마다 별도의 JSON 파일에 동작인식 결과 저장|

이렇게 서로 다른 위치에 있는 파일을 하나씩 찾는 대신, Python 코드로 자동으로 수집하겠습니다.

**이번 코드의 처리 흐름**

```
교과 9 OCR 검사 기록 읽기
            +
교과 10 동작인식 결과 파일들 읽기
            ↓
JSON 파일과 내용 확인
            ↓
교과 13에 두 파일로 정리하여 저장
 ├─ inspection_history.json (OCR 검사 기록)
 └─ action_jobs.json        (동작인식 결과 목록)
            ↓
가져온 기록 개수 출력
```

기존 교과의 원본 파일은 수정하지 않습니다. **이번 실습은 결과를 가져오는 단계이며, 공통** `**InferenceEvent**` **형식으로 변환하는 작업은 다음 실습에서 진행합니다.**

아래 코드를 작성합니다. 기존 교과 프로젝트의 위치가 다르다면 `SUBJECT9`, `SUBJECT10` 경로를 수정합니다.

이전 교과의 결과 파일이 없는 학생은 강사가 제공한 `handoff/provided/`의 파일을 자동으로 사용합니다.

**작성할 파일:** `scripts/01_import_results.py`

```python

"""교과 9·10에서 이미 저장한 JSON 결과를 교과 13으로 가져옵니다."""

import json
from pathlib import Path


# ============================================================
# [1단계: 입력 경로 준비]
# 이전 교과의 JSON 파일 위치와 교과 13의 저장 위치를 정합니다.
# ============================================================

# 현재 교과 13 프로젝트의 최상위 폴더
ROOT = Path(__file__).resolve().parents[1]

# 현재 사용자 홈 폴더 (예: /home/youjung)
HOME = Path.home()

# 교과 9·10에서 실제로 실습했던 프로젝트 폴더
SUBJECT9 = HOME / "ai_vision" / "subject9_machinevision"
SUBJECT10 = HOME / "ai_vision" / "subject10_pose_action"

# 이전 교과 결과가 없을 때 강사가 제공한 파일을 찾을 폴더
PROVIDED9 = ROOT / "handoff" / "provided" / "subject9"
PROVIDED10 = ROOT / "handoff" / "provided" / "subject10"

# 교과 13에서 가져온 결과를 저장할 폴더
OUTPUT = ROOT / "handoff" / "records"

# 교과 9: 여러 OCR 검사 기록이 담긴 JSON 파일
HISTORY = Path("reports/day10/service_history.json")

# 교과 10: 각 작업ID 폴더에 저장된 동작인식 결과 JSON
# * 는 서로 다른 작업ID 폴더 이름을 뜻합니다.
JOB_RESULTS = "reports/day10/web/jobs/*/result.json"


# ============================================================
# [2단계: 입력 파일 찾기]
# 기존 프로젝트를 먼저 확인하고, 없으면 강사 제공 폴더를 찾습니다.
# ============================================================

def choose_history() -> tuple[Path, Path]:
    """교과 9의 OCR 검사 이력 파일 1개를 찾습니다."""

    # 실제 교과 9 프로젝트 → 강사 제공 자료 순서로 확인
    for project in (SUBJECT9, PROVIDED9):
        path = project / HISTORY

        if path.is_file():
            # 파일을 찾으면 프로젝트 위치와 파일 경로 반환
            return project, path

    # 두 위치 모두 파일이 없다면 오류 발생
    raise FileNotFoundError(
        "교과 9 검사 기록을 찾을 수 없습니다. "
        "기존 프로젝트의 reports/day10/service_history.json을 확인하거나 "
        "handoff/provided/subject9/에 같은 경로로 배치하세요."
    )


def choose_jobs() -> tuple[Path, list[Path]]:
    """교과 10의 동작인식 결과 파일들을 찾습니다."""

    # 실제 교과 10 프로젝트 → 강사 제공 자료 순서로 확인
    for project in (SUBJECT10, PROVIDED10):

        # jobs 아래 여러 작업ID 폴더에서 result.json 검색
        found = sorted(project.glob(JOB_RESULTS))

        if found:
            # 파일을 찾으면 프로젝트 위치와 파일 목록 반환
            return project, found

    # 두 위치 모두 결과 파일이 없다면 오류 발생
    raise FileNotFoundError(
        "교과 10의 jobs/*/result.json을 찾을 수 없습니다. "
        "기존 프로젝트 경로 또는 handoff/provided/subject10/을 확인하세요."
    )


# ============================================================
# [3단계: JSON 읽기]
# 저장된 JSON 파일의 내용을 Python 데이터로 변환합니다.
# ============================================================

def read_json(path: Path):
    """JSON 파일을 읽어 Python List 또는 Dictionary로 반환합니다."""

    # utf-8-sig는 일반 UTF-8과 BOM이 있는 UTF-8 파일을 읽을 수 있습니다.
    return json.loads(
        path.read_text(encoding="utf-8-sig")
    )


# ============================================================
# [4단계: 입력 파일 읽기 및 교과 9 기본 구조 검사]
# 기존 결과를 찾고 교과 9의 OCR 검사 기록을 읽습니다.
# ============================================================

def main() -> None:

    # 교과 9 검사 기록 파일 찾기
    origin9, history_path = choose_history()

    # 교과 10 동작인식 결과 파일들 찾기
    origin10, job_files = choose_jobs()

    # 교과 9 JSON 파일 읽기
    inspection = read_json(history_path)

    # 교과 9는 여러 검사 기록이 들어 있는 List여야 합니다.
    # 또한 최소 1건 이상의 기록이 있어야 합니다.
    if not isinstance(inspection, list) or not inspection:
        raise ValueError(
            "교과 9의 service_history.json은 "
            "1건 이상의 JSON 배열이어야 합니다."
        )


    # ========================================================
    # [5단계: 교과 10 결과 검사 및 목록 만들기]
    # 작업별 result.json을 읽어 하나의 List로 정리합니다.
    # ========================================================

    # 동작인식 결과를 모아 둘 빈 List
    actions = []

    # 찾은 result.json 파일들을 하나씩 처리
    for path in job_files:

        # 현재 작업의 JSON 읽기
        result = read_json(path)

        # JSON이 Dictionary이고 job_id가 있는지 확인
        if not isinstance(result, dict) or not result.get("job_id"):
            raise ValueError(
                f"교과 10 결과에 job_id가 없습니다: {path}"
            )

        # 원본 파일 경로와 실제 결과를 함께 저장
        actions.append(
            {
                "source_path": path.relative_to(origin10).as_posix(),
                "result": result
            }
        )


    # ========================================================
    # [6단계: 교과 9 세부 기록 검사]
    # OCR 검사 기록마다 고유한 request_id가 있는지 확인합니다.
    # ========================================================

    # 모든 검사 기록이 Dictionary이며 request_id를 가지고 있는지 검사
    if not all(
        isinstance(row, dict) and row.get("request_id")
        for row in inspection
    ):
        raise ValueError(
            "교과 9 검사 기록에 request_id가 없는 항목이 있습니다."
        )


    # ========================================================
    # [7단계: 출력 파일 저장 및 결과 확인]
    # 두 교과의 결과를 교과 13의 handoff/records/에 저장합니다.
    # ========================================================

    # 저장 폴더가 없다면 자동 생성
    OUTPUT.mkdir(parents=True, exist_ok=True)

    # 교과 9의 OCR 검사 기록 저장
    (OUTPUT / "inspection_history.json").write_text(
        json.dumps(inspection, indent=2, ensure_ascii=False),
        encoding="utf-8"
    )

    # 교과 10의 동작인식 결과 목록 저장
    (OUTPUT / "action_jobs.json").write_text(
        json.dumps(actions, indent=2, ensure_ascii=False),
        encoding="utf-8"
    )

    # 터미널에 실제 가져온 결과 개수 출력
    print(f"교과 9 OCR 기록: {len(inspection)}건")
    print(f"교과 10 Action 기록: {len(actions)}건")

    # 저장 완료 안내
    print("완료: handoff/records/inspection_history.json, action_jobs.json")
    print("참고: 저장된 결과를 읽었으며 AI 모델을 새로 실행하지 않았습니다.")


# ============================================================
# [8단계: 프로그램 실행]
# 이 파일을 직접 실행했을 때 main() 함수를 시작합니다.
# ============================================================

if __name__ == "__main__":
    main()

```

#### 의사코드

```
[1단계] 입력 경로 준비
────────────────────────────────────
현재 교과 13 프로젝트 위치 확인
        ↓
교과 9 프로젝트 경로 설정
→ subject9_machinevision
        ↓
교과 10 프로젝트 경로 설정
→ subject10_pose_action
        ↓
강사 제공 자료 경로 설정
→ handoff/provided/
        ↓
결과 저장 경로 설정
→ handoff/records/


[2단계] 이전 교과의 JSON 파일 찾기
────────────────────────────────────
교과 9 프로젝트에서
service_history.json 검색
        ↓
파일이 없으면
→ 강사 제공 subject9 폴더에서 검색
        ↓
교과 10 프로젝트에서
jobs/*/result.json 검색
        ↓
파일이 없으면
→ 강사 제공 subject10 폴더에서 검색
        ↓
필요한 파일을 찾지 못하면
→ 오류 메시지 출력 후 중단


[3단계] JSON 파일 읽기
────────────────────────────────────
찾은 JSON 파일 열기
        ↓
JSON 내용을 Python에서 사용할 수 있는
List 또는 Dictionary로 변환


[4단계] 교과 9 OCR 기록 읽기 및 기본 검사
────────────────────────────────────
교과 9 service_history.json 읽기
        ↓
여러 검사 기록이 담긴 List인지 확인
        ↓
검사 기록이 1건 이상 있는지 확인
        ↓
조건을 만족하지 않으면 오류 발생


[5단계] 교과 10 동작인식 결과 정리
────────────────────────────────────
교과 10의 result.json 파일들을 하나씩 읽기
        ↓
각 JSON이 Dictionary인지 확인
        ↓
각 결과에 job_id가 있는지 확인
        ↓
원본 파일의 상대경로와 검사 결과를 묶기
        ↓
actions List에 추가
        ↓
모든 작업의 result.json 처리 완료


[6단계] 교과 9 세부 기록 검사
────────────────────────────────────
교과 9 OCR 검사 기록을 하나씩 확인
        ↓
각 기록이 Dictionary인지 확인
        ↓
각 기록에 request_id가 있는지 확인
        ↓
하나라도 조건을 만족하지 않으면
→ 오류 발생 후 중단


[7단계] 교과 13에 결과 저장 및 출력
────────────────────────────────────
handoff/records/ 폴더 생성
        ↓
교과 9 OCR 검사 기록 저장
→ inspection_history.json
        ↓
교과 10 동작인식 결과 목록 저장
→ action_jobs.json
        ↓
교과 9에서 가져온 검사 건수 출력
        ↓
교과 10에서 가져온 작업 건수 출력
        ↓
저장 완료 메시지 출력


[8단계] 프로그램 실행
────────────────────────────────────
python scripts/01_import_results.py
        ↓
main() 함수 실행
        ↓
1~7단계 처리 수행
        ↓
정상 완료 시 결과 파일 2개 생성
        ↓
프로그램 종료
```

#### 코드를 실행합니다

```bash
python scripts/01_import_results.py
ls -lh handoff/records/
```

앞서 검증한 예시 자료를 사용했다면 다음처럼 출력됩니다.

```text
교과 9 OCR 기록: 5건
교과 10 Action 기록: 6건
완료: handoff/records/inspection_history.json, action_jobs.json
참고: 저장된 결과를 읽었으며 AI 모델을 새로 실행하지 않았습니다.
```

학생마다 기존에 만든 결과 건수는 다를 수 있습니다. **개수가 다르다는 이유만으로 오류는 아닙니다.** 다음 두 파일이 생성되고 각 파일에 결과가 1건 이상 들어 있는지 확인합니다.

```text
handoff/records/
├── inspection_history.json
└── action_jobs.json
```

### 실습 7. 교과 9와 교과 10의 JSON 결과를 비교합니다

앞에서 교과 9와 교과 10의 JSON 파일을 교과 13으로 가져왔습니다.

이번에는 **두 교과가 검사 결과를 어떤 이름으로 저장했는지** 확인하겠습니다.

별도의 Python 파일을 만들지 않고 터미널에서 아래 코드를 실행합니다.

```bash
python - <<'PY'
import json
from pathlib import Path

base = Path("handoff/records")

inspection = json.loads(
    (base / "inspection_history.json").read_text(encoding="utf-8")
)

actions = json.loads(
    (base / "action_jobs.json").read_text(encoding="utf-8")
)

print("교과 9 주요 필드:", list(inspection[0].keys()))
print("교과 10 주요 필드:", list(actions[0]["result"].keys()))
PY
```

### 실제 실행 결과

**① 교과 9 — OCR 검사 결과**

```bash
교과 9 주요 필드:
['request_id', 'inspected_at', 'service_version',
 'inspection_module', 'rule_version', 'file_name',
 'measurements', 'final_decision', 'reasons']
```

교과 9에서는 OCR 검사를 수행한 이미지와 검사 판정을 기록합니다.

| 필드 | 쉽게 이해하기 |
|---|---|
| `file_name` | 어떤 제품 이미지를 검사했나요? |
| `measurements` | OCR이 읽은 문자와 기준 문자는 무엇인가요? |
| `final_decision` | 검사 결과가 OK인가요, NG인가요? |
| `reasons` | NG라면 그 이유는 무엇인가요? |

**② 교과 10 — 동작인식 결과**

```bash
교과 10 주요 필드:
['job_id', 'created_at', 'source_file',
 'expected_action', 'expected_action_description',
 'detected_action', 'detected_action_description',
 'latest_confidence', 'final_match', 'metrics',
 'runtime', 'runtime_csv', ...]
```

위 출력은 실제 확인된 항목 중 일부입니다.

교과 10에서는 영상에서 예측한 동작과 기대 동작을 비교한 결과를 기록합니다.

| 필드 | 쉽게 이해하기 |
|---|---|
| `source_file` | 어떤 영상을 분석했나요? |
| `expected_action` | 원래 어떤 동작이어야 하나요? |
| `detected_action` | AI가 어떤 동작이라고 예측했나요? |
| `latest_confidence` | 마지막 예측의 신뢰도는 얼마인가요? |
| `final_match` | 예측 동작과 기대 동작이 일치하나요? |

### 결과를 어떻게 이해하면 될까요?

두 교과 모두 AI가 분석한 결과를 JSON으로 저장하지만, **검사하는 대상이 달라 필드 이름도 서로 다릅니다.**

```text
교과 9 — 제품 OCR 검사
→ 문자를 읽고 OK / NG 판단
→ final_decision에 기록

교과 10 — 사람의 동작인식
→ 동작을 예측하고 기대 동작과 비교
→ detected_action, final_match에 기록
```

지금 실행한 코드는 두 JSON의 **첫 번째 기록에서 필드 이름만 확인**한 것입니다. 실제 검사나 동작인식을 다시 수행하지 않았습니다.

---

## PART 3. 두 결과를 공통 JSON으로 변환합니다

### 실습 8. 공통 JSON 데이터 모델을 작성합니다

앞에서 교과 9의 OCR 검사 결과와 교과 10의 동작인식 결과를 가져왔습니다.

그런데 두 교과는 같은 의미의 정보도 서로 다른 이름으로 저장하고 있습니다.

예를 들어 다음과 같습니다.

- **교과 9:** `file_name` → 검사한 이미지 이름
    
- **교과 10:** `source_file` → 분석한 영상 이름
    

두 항목은 모두 **어떤 파일을 검사했는가?**라는 의미이므로, 교과 13에서는 `input_ref`라는 하나의 이름으로 통일하려고 합니다.

이것은 서로 다른 양식으로 작성된 두 보고서를 **하나의 공통 보고서 양식으로 정리하는 것**과 같습니다.

이때 사용할 공통 양식이 `InferenceEvent`입니다.

이번 실습에서는 **Pydantic**을 사용하여 `InferenceEvent` 클래스를 작성합니다. Pydantic은 데이터에 필요한 항목과 자료형을 정하고, 값이 올바른 형식인지 검사하는 도구입니다.

즉, 이번에는 JSON 데이터를 실제로 변환하기 전에 **두 교과의 결과를 어떤 공통 형식으로 저장할지 먼저 정하는 단계**입니다.

**작성할 파일:** `app/schemas.py`

```python
from datetime import datetime
from typing import Any, Literal

from pydantic import BaseModel, Field


class InferenceEvent(BaseModel):
    """교과 9·10 과거 결과를 공통 형식으로 표현합니다."""

    event_id: str
    task: Literal["inspection", "action"]
    source_mode: Literal["recorded_result"] = "recorded_result"
    source_project: str
    source_path: str
    recorded_at: datetime
    input_ref: str
    processor_id: str
    processor_version: str | None = None
    predicted_label: str | None = None
    expected_label: str | None = None
    confidence: float | None = Field(default=None, ge=0.0, le=1.0)
    decision: Literal["OK", "NG", "UNKNOWN"]
    reasons: list[str] = Field(default_factory=list)
    details: dict[str, Any] = Field(default_factory=dict)
```

#### `InferenceEvent`는 어떤 역할을 하나요?

`InferenceEvent`는 데이터를 확인하는 **공통 검사표**라고 생각하면 됩니다.

예를 들어 학교에서 학생마다 제출한 보고서의 양식이 다르면 관리하기 어렵습니다. 그래서 학교는 이름, 학번, 제출일처럼 반드시 기록해야 할 항목과 작성 규칙을 정합니다.

교과 13의 `InferenceEvent`도 같은 역할을 합니다.

교과 9·10에서 가져온 결과가 우리가 정한 공통 형식에 맞는지 검사합니다.

##### 1. 어떤 데이터를 검사하나요?

앞 실습에서 저장한 두 파일이 있습니다.

```
handoff/records/
├── inspection_history.json  ← 교과 9 OCR 결과
└── action_jobs.json         ← 교과 10 동작인식 결과
```

이 파일의 데이터를 **다음 실습에서 공통 형식으로 변환할 때**, `InferenceEvent`를 사용하여 검증합니다.

```
교과 9·10에서 가져온 JSON
          ↓
공통 필드 이름으로 변환
          ↓
InferenceEvent
          ↓
필수 항목·자료형·허용값 검사
          ↓
검사 통과 → 공통 JSON 생성
검사 실패 → 오류 발생
```

**주의:** 지금 작성한 `schemas.py`는 검사 규칙만 정의합니다. 이 파일만 작성한다고 JSON이 자동으로 변환되지는 않습니다.

##### 2. 데이터 타입은 어떤 기준으로 정했나요?

실제 교과 9·10 JSON에 저장된 값과 각 항목의 의미를 기준으로 정했습니다.

|공통 필드|기존 JSON에서 가져올 값|Python 자료형|
|---|---|---|
|`input_ref`|이미지 파일명 또는 영상 파일명|`str` (문자열)|
|`recorded_at`|`inspected_at` 또는 `created_at`|`datetime` (날짜·시간)|
|`predicted_label`|OCR 문자열 또는 예측 동작|`str \| None`|
|`confidence`|예측 신뢰도, 없으면 `None`|`float \| None`|
|`decision`|최종 검사 판정|`OK`, `NG`, `UNKNOWN` 중 하나|
|`reasons`|판정 이유 목록|`list[str]`|
|`details`|OCR 측정값 등 교과별 상세 정보|`dict` (딕셔너리)|

예를 들어 다음 코드는:

```python
confidence: float | None = Field(
    default=None,
    ge=0.0,
    le=1.0
)
```

신뢰도가 **0.0부터 1.0 사이의 숫자이거나** `**None**`**(값 없음)** 이어야 한다는 규칙입니다.

교과 9에는 신뢰도 값이 없으므로 `None`을 사용하고, JSON으로 저장할 때는 `null`로 표현합니다.

또한 `event_id`, `source_project`, `source_path` 등은 교과 13에서 결과의 고유번호와 출처를 관리하기 위한 항목입니다.

##### 3. 실제로 데이터 검증을 테스트해 봅니다

아직 FastAPI 서버를 실행할 필요는 없습니다. Python에서 `InferenceEvent`를 직접 호출하여 정상 데이터와 잘못된 데이터를 검사하겠습니다.

교과 13 프로젝트 최상위 폴더에서 실행합니다.

```bash
python - <<'PY'
import json
from pathlib import Path
from pydantic import ValidationError
from app.schemas import InferenceEvent

# 1. 교과 9에서 가져온 실제 OCR 기록 1건 읽기
path = Path("handoff/records/inspection_history.json")
history = json.loads(path.read_text(encoding="utf-8"))
row = history[0]

# 2. 실제 OCR 기록을 공통 필드 이름으로 정리
data = {
    "event_id": f"inspection-{row['request_id']}",
    "task": "inspection",
    "source_project": "subject9_machinevision",
    "source_path": "reports/day10/service_history.json",
    "recorded_at": row["inspected_at"],
    "input_ref": row["file_name"],
    "processor_id": row["inspection_module"],
    "processor_version": row["rule_version"],
    "predicted_label": row["measurements"]["ocr_text"],
    "expected_label": row["measurements"]["expected_text"],
    "confidence": None,
    "decision": row["final_decision"],
    "reasons": row["reasons"],
    "details": row["measurements"],
}

# 3. 정상 데이터인지 검사
event = InferenceEvent.model_validate(data)
print("정상 검증: 통과")
print("검사 결과:", event.decision)
print("날짜 자료형:", type(event.recorded_at).__name__)
print("JSON 표현:", json.dumps({"confidence": event.confidence}))

# 4. 일부러 잘못된 신뢰도 1.5를 넣어 검사
try:
    InferenceEvent.model_validate({**data, "confidence": 1.5})
except ValidationError:
    print("잘못된 데이터 검증: 오류 감지 성공")
PY
```

`python - <<'PY'`는 별도의 Python 파일을 만들지 않고 터미널에서 코드를 바로 실행하는 방법입니다.

##### 4. 실행 결과를 확인합니다

정상적으로 실행하면 다음과 같은 결과가 나옵니다.

```
정상 검증: 통과
검사 결과: NG
날짜 자료형: datetime
JSON 표현: {"confidence": null}
잘못된 데이터 검증: 오류 감지 성공
```

`검사 결과`는 실제로 읽은 첫 번째 OCR 기록에 따라 `OK` 또는 `NG`로 달라집니다.

**결과 해석**

- `정상 검증: 통과` → 입력 데이터가 공통 형식에 맞습니다.
    
- `날짜 자료형: datetime` → 날짜 문자열이 Python의 날짜·시간 자료형으로 변환되었습니다.
    
- `"confidence": null` → 신뢰도 값이 없다는 뜻입니다.
    
- `오류 감지 성공` → 허용 범위를 벗어난 `1.5`를 Pydantic이 찾아냈습니다.
    

**오늘의 핵심:** `InferenceEvent`는 교과 9·10에서 가져온 데이터를 공통 JSON으로 정리할 때, 필요한 항목이 있는지와 값의 형식이 올바른지를 확인하는 검사표입니다.

이번 테스트에서는 교과 9의 실제 기록 1건으로 검사 기능을 확인했습니다. 이후 실습에서 교과 9와 교과 10의 전체 결과를 변환할 때 같은 규칙을 적용합니다.

---
### 실습 9. 교과 9·10의 검사 결과를 공통 JSON으로 변환합니다

앞에서 교과 9와 교과 10의 검사 결과가 서로 다른 JSON 형식으로 저장되어 있다는 것을 확인했습니다.

이번에 작성할 `app/records.py`**는 서로 다른 두 JSON 결과를 교과 13에서 사용할 공통 형식(**`InferenceEvent`**)으로 바꿔주는 파일**입니다.

쉽게 말하면, 서로 다른 양식으로 작성된 두 보고서를 **하나의 공통 보고서 양식으로 정리하는 
프로그램**입니다.

예를 들어 교과 9의 `final_decision`은 공통 필드인 `decision`으로 옮기고, 
교과 10은 `detected_action`과 `final_match`를 확인하여 공통 판정값을 만듭니다.

이때 기존 JSON은 수정하지 않고, 필요한 정보를 읽어 새로운 형태로 변환합니다.

```
교과 9 OCR 검사 결과       교과 10 동작인식 결과
         ↓                         ↓
 inspection_to_event()      action_to_event()
         ↓                         ↓
         └──────────┬──────────────┘
                    ↓
          공통 JSON 형식으로 변환
              InferenceEvent
                    ↓
           FastAPI에서 조회·활용
```

이 파일에는 두 가지 변환 함수를 작성합니다.

- `inspection_to_event()` : 교과 9의 OCR 검사 결과를 공통 JSON으로 변환합니다.
    
- `action_to_event()` : 교과 10의 동작인식 결과를 공통 JSON으로 변환합니다.
    

**핵심:** `app/records.py`는 AI 모델을 다시 실행하는 파일이 아니라, 
이미 만들어진 검사 결과를 하나의 공통 형식으로 정리하는 파일입니다.

**파일작성:** `app/records.py`

```python
"""교과 9·10의 실제 저장 결과를 공통 이벤트로 변환합니다."""

import json
from pathlib import Path

from app.schemas import InferenceEvent

ROOT = Path(__file__).resolve().parents[1]
RECORDS = ROOT / "handoff" / "records"
INSPECTION_FILE = RECORDS / "inspection_history.json"
ACTION_FILE = RECORDS / "action_jobs.json"


def _read_json(path: Path):
    if not path.is_file():
        raise FileNotFoundError(
            f"필수 데이터가 없습니다: {path.name}. "
            "먼저 python scripts/01_import_results.py 를 실행하세요."
        )
    return json.loads(path.read_text(encoding="utf-8"))


def inspection_to_event(row: dict) -> InferenceEvent:
    """교과 9의 OCR 문자열 판정을 변경 없이 옮깁니다."""
    measurements = row["measurements"]
    return InferenceEvent(
        event_id="inspection:" + row["request_id"],
        task="inspection",
        source_project="subject9_openCV",
        source_path="reports/day10/service_history.json",
        recorded_at=row["inspected_at"],
        input_ref=row["file_name"],
        processor_id=row["inspection_module"],
        processor_version=row["rule_version"],
        predicted_label=measurements["ocr_text"],
        expected_label=measurements["expected_text"],
        confidence=None,  # 규칙 기반 OCR JSON에 확률 점수가 없습니다.
        decision=row["final_decision"],
        reasons=row["reasons"],
        details={
            "service_version": row["service_version"],
            "pattern_ok": measurements["pattern_ok"],
            "expected_match": measurements["expected_match"],
        },
    )


def action_to_event(item: dict) -> InferenceEvent:
    """UNKNOWN/NO_PREDICTION을 단순 NG로 잘못 해석하지 않습니다."""
    row = item["result"]
    detected = row.get("detected_action")
    final_match = row.get("final_match")

    if detected in (None, "", "UNKNOWN", "NO_PREDICTION"):
        decision = "UNKNOWN"
        reasons = ["동작을 확정하지 못했습니다."]
    elif final_match is True:
        decision = "OK"
        reasons = []
    elif final_match is False:
        decision = "NG"
        reasons = ["예측한 동작이 기대 동작과 다릅니다."]
    else:
        decision = "UNKNOWN"
        reasons = ["최종 비교 결과가 기록되지 않았습니다."]

    return InferenceEvent(
        event_id="action:" + row["job_id"],
        task="action",
        source_project="subject10_pose_action",
        source_path=item["source_path"],
        recorded_at=row["created_at"],
        input_ref=row["source_file"],
        processor_id="pose_action_pipeline",
        processor_version=None,  # 원본 결과 JSON에 버전 값이 없습니다.
        predicted_label=detected,
        expected_label=row.get("expected_action"),
        confidence=row.get("latest_confidence"),
        decision=decision,
        reasons=reasons,
        details={
            "final_match": final_match,
            "metrics": row.get("metrics", {}),
            "runtime": row.get("runtime", {}),
        },
    )


def load_events() -> list[InferenceEvent]:
    inspections = _read_json(INSPECTION_FILE)
    actions = _read_json(ACTION_FILE)
    events = [inspection_to_event(row) for row in inspections]
    events.extend(action_to_event(item) for item in actions)
    # 기존 OCR 데이터의 기록 시각에는 시간대 정보가 없으므로
    # UTC로 임의 변환하지 않고 원본의 시각을 그대로 보존합니다.
    events.sort(
        key=lambda event: event.recorded_at.strftime("%Y-%m-%dT%H:%M:%S"),
        reverse=True,
    )
    return events
```

#### 코드 이해하기

`inspection_to_event()`는 OCR 검사 결과의 문자열과 NG 이유를 보존합니다. OCR에는 신뢰도 필드가 없으므로 `confidence=None`을 저장합니다.

`action_to_event()`는 `NO_PREDICTION` 또는 `UNKNOWN`인 경우 `UNKNOWN`으로 표시합니다. `final_match=False`라도 동작이 확정되지 않았으면 **NG로 단정하지 않습니다.**

`load_events()`는 두 결과를 하나의 리스트로 합칩니다. 기록 시간에 시간대 정보가 포함된 경우와 그렇지 않은 경우가 섞여 있으므로, 1일차에는 **기록된 문자 형태의 날짜·시간을 비교해 목록으로 보여주는 수준**으로 사용합니다. 실제 시간대 기준 정렬은 2일차 이후 데이터 설계 단계에서 보완합니다.

### 실습 10. FastAPI 실행 전에 데이터 변환을 먼저 확인합니다

문제가 생겼을 때 어떤 단계에서 실패했는지 알 수 있도록, 서버 없이 데이터를 먼저 읽어 봅니다.

```bash
python - <<'PY'
from app.records import load_events
events = load_events()
print("전체:", len(events))
print("교과 9:", sum(event.task == "inspection" for event in events))
print("교과 10:", sum(event.task == "action" for event in events))
for event in events[:3]:
    print(event.task, event.predicted_label, event.decision, event.confidence)
PY
```

예시 자료를 사용하면 `전체: 11`, `교과 9: 5`, `교과 10: 6`이 출력됩니다. 학생마다 기록 개수는 다를 수 있습니다. **두 교과가 각각 1건 이상 출력되고 오류가 나지 않으면** 다음 단계로 진행합니다.

---

## PART 4. FastAPI로 공통 기록을 조회합니다

### 실습 11. FastAPI 서버 파일을 작성합니다

#### 1. 이번 실습에서는 무엇을 하나요?

앞에서는 교과 9의 OCR 검사 결과와 교과 10의 동작인식 결과를 `InferenceEvent`라는 
공통 형식으로 변환했습니다.

하지만 이렇게 정리한 결과를 웹브라우저에서 확인하려면 **필요한 데이터를 요청받아 전달해 주는 프로그램**이 있어야 합니다.

이번에는 **FastAPI를 사용하여 검사 결과를 요청받고 전달하는 서버 프로그램**을 작성하겠습니다.

#### 2. FastAPI는 어떤 역할을 하나요?

쉽게 비유하면 FastAPI는 **도서관의 안내 직원**과 같습니다.

도서관에서 원하는 책을 요청하면 안내 직원이 책을 찾아 전달해 주듯이, FastAPI는 사용자가 원하는 검사 기록을 찾아 JSON 데이터로 전달합니다.

예를 들어 다음과 같습니다.

- 전체 검사 기록을 요청하면 → 교과 9·10의 결과 목록을 전달합니다.
    
- OCR 결과만 요청하면 → 교과 9의 검사 기록만 전달합니다.
    
- 동작인식 결과만 요청하면 → 교과 10의 검사 기록만 전달합니다.
    
- 특정 기록을 요청하면 → 해당 기록의 자세한 내용을 전달합니다.
    

이렇게 **프로그램이 데이터를 요청하고 전달받을 수 있도록 만든 통로를 API**라고 합니다.

#### 3. GET API란 무엇인가요?

`GET`은 서버에 저장된 정보를 요청하여 가져올 때 사용하는 HTTP 요청 방식입니다.

이번 실습에서는 이미 저장된 JSON 결과를 조회하기 때문에 `GET`을 사용합니다.

아직 새로운 이미지를 OCR로 검사하거나 영상을 동작인식 모델에 입력하지는 않습니다. 실제 AI 모델 실행 기능은 4일차에 연결합니다.

#### 4. 이번에 만들 API 주소와 역할

|API 주소|어떤 기능인가요?|
|---|---|
|`GET /`|웹 조회 화면을 보여줍니다. 화면 파일은 다음 실습에서 작성합니다.|
|`GET /health`|서버가 응답하는지와 결과 JSON 파일이 준비되어 있는지 확인합니다.|
|`GET /api/v1/events`|교과 9와 10의 전체 검사 기록을 조회합니다.|
|`GET /api/v1/events?task=inspection`|교과 9의 OCR 검사 기록만 조회합니다.|
|`GET /api/v1/events?task=action`|교과 10의 동작인식 기록만 조회합니다.|
|`GET /api/v1/summary`|전체 기록 수와 OK·NG·UNKNOWN 등의 통계를 조회합니다.|
|`GET /api/v1/events/{event_id}`|고유번호를 이용해 특정 검사 기록 한 건을 조회합니다.|

여기서 `task=inspection`과 `task=action`은 **어떤 검사 결과를 보고 싶은지 선택하는 검색 조건**입니다.

`{event_id}`는 여러 검사 기록 중 원하는 기록 한 건을 찾기 위한 고유번호입니다.

#### 5. 프로그램의 전체 처리 흐름

```
사용자가 웹브라우저에서 결과 요청
                ↓
          FastAPI 서버
                ↓
       어떤 결과를 요청했는지 확인
                ↓
       공통 InferenceEvent 읽기
                ↓
         필요한 결과만 선택
                ↓
          JSON 데이터 전달
                ↓
       웹브라우저에 결과 표시
```

이번에 작성할 `app/main.py`는 FastAPI 서버의 중심 파일입니다.

앞에서 작성한 `app/records.py`의 `load_events()` 함수를 이용하여 공통 검사 결과를 가져오고, 사용자가 요청한 주소에 따라 필요한 결과를 전달하도록 작성합니다.

**핵심:** 지금은 AI 모델을 새로 실행하는 것이 아니라, **이전에 저장한 OCR·동작인식 결과를 FastAPI를 통해 조회할 수 있도록 만드는 단계**입니다. 다음 실습에서는 이 API와 웹 화면을 연결하겠습니다.

**파일: `app/main.py`**

```python
"""
교과 13 · 1일차 FastAPI 서버

교과 9의 OCR 검사 결과와 교과 10의 동작인식 결과를
공통 InferenceEvent 형식으로 읽어 웹브라우저에 전달합니다.

오늘은 기존 결과를 조회하며 AI 모델을 다시 실행하지 않습니다.
"""


# ============================================================
# [1] 필요한 라이브러리와 기존 코드 가져오기
# FastAPI 서버를 만들고 이전 실습의 데이터를 사용하기 위한 준비
# ============================================================

from pathlib import Path
from typing import Literal

# FastAPI: 웹 API를 만드는 도구
# HTTPException: 요청을 처리할 수 없을 때 오류 응답을 보내는 도구
# Query: 사용자가 입력한 검색 조건을 검사하는 도구
from fastapi import FastAPI, HTTPException, Query

# HTML 파일을 웹브라우저에 전달하는 도구
from fastapi.responses import FileResponse

# 앞에서 작성한 JSON 결과 파일 경로와 데이터 불러오기 함수
from app.records import (
    ACTION_FILE,
    INSPECTION_FILE,
    load_events,
)

# 공통 JSON 데이터 규격
from app.schemas import InferenceEvent


# ============================================================
# [2] FastAPI 서버와 웹 화면 경로 설정
# 서버의 이름과 웹브라우저에 보여줄 HTML 파일 위치 지정
# ============================================================

# FastAPI 프로그램 생성
app = FastAPI(
    title="교과 13 공통 AI 플랫폼",
    version="0.1.0"
)

# 웹 화면 파일(index.html)의 실제 위치 지정
INDEX_PAGE = (
    Path(__file__).resolve().parents[1]
    / "web"
    / "index.html"
)


# ============================================================
# [3] 공통 검사 결과 불러오기
# 교과 9·10의 결과를 읽고 문제가 있으면 오류 메시지 반환
# ============================================================

def ready_events() -> list[InferenceEvent]:

    try:
        # 기존 JSON 결과를 공통 이벤트 목록으로 가져오기
        return load_events()

    except (FileNotFoundError, ValueError, KeyError) as exc:

        # 파일이 없거나 데이터 형식에 문제가 있으면
        # HTTP 503 오류로 알려주기
        raise HTTPException(
            status_code=503,
            detail=str(exc)
        ) from exc


# ============================================================
# [4] 메인 웹 화면 제공
# 사용자가 기본 주소에 접속하면 index.html을 전달
# ============================================================

@app.get("/")
def home():

    # web/index.html 파일을 브라우저에 전달
    return FileResponse(INDEX_PAGE)


# ============================================================
# [5] 서버와 결과 파일 준비 상태 확인
# 서버가 응답하는지, 필요한 JSON 파일이 존재하는지 검사
# ============================================================

@app.get("/health")
def health():

    return {
        # 서버 응답 상태
        "status": "ok",

        # 이전에 저장된 AI 결과를 사용하는 방식
        "mode": "recorded_result",

        # 교과 9와 교과 10의 JSON 파일이 모두 있는지 확인
        "data_ready": (
            INSPECTION_FILE.is_file()
            and ACTION_FILE.is_file()
        ),
    }


# ============================================================
# [6] 검사 결과 목록 조회
# 전체 기록 또는 선택한 교과의 검사 결과만 전달
# ============================================================

@app.get(
    "/api/v1/events",
    response_model=list[InferenceEvent]
)
def get_events(

    # inspection: 교과 9 OCR
    # action: 교과 10 동작인식
    # None: 교과 구분 없이 전체 조회
    task: Literal["inspection", "action"] | None = None,

    # 한 번에 가져올 기록 개수
    # 기본 30건, 최소 1건, 최대 100건
    limit: int = Query(default=30, ge=1, le=100),
):

    # 공통 검사 기록 불러오기
    events = ready_events()

    # 특정 교과를 선택했다면 해당 결과만 남기기
    if task is not None:

        events = [
            event
            for event in events
            if event.task == task
        ]

    # 요청한 개수만큼 결과 반환
    return events[:limit]


# ============================================================
# [7] 검사 결과 요약 통계 조회
# 전체 기록 수, 교과별 기록 수, 판정별 기록 수 계산
# ============================================================

@app.get("/api/v1/summary")
def get_summary():

    # 공통 검사 기록 불러오기
    events = ready_events()

    return {

        # 전체 검사 기록 개수
        "total": len(events),

        # 교과 9 OCR / 교과 10 Action 기록 개수
        "by_task": {
            name: sum(
                event.task == name
                for event in events
            )
            for name in ("inspection", "action")
        },

        # OK / NG / UNKNOWN 판정별 개수
        "by_decision": {
            name: sum(
                event.decision == name
                for event in events
            )
            for name in ("OK", "NG", "UNKNOWN")
        },

        # 저장된 과거 결과를 조회하는 방식
        "mode": "recorded_result",
    }


# ============================================================
# [8] 특정 검사 기록 한 건 조회
# event_id를 이용하여 원하는 검사 결과의 상세 내용 전달
# ============================================================

@app.get(
    "/api/v1/events/{event_id}",
    response_model=InferenceEvent
)
def get_event(event_id: str):

    # 전체 검사 기록에서 요청한 ID 검색
    for event in ready_events():

        # 고유번호가 일치하는 기록을 찾았다면 반환
        if event.event_id == event_id:
            return event

    # 해당 ID가 없다면 HTTP 404 오류 반환
    raise HTTPException(
        status_code=404,
        detail="이벤트를 찾을 수 없습니다."
    )
```

#### 의사코드

```
[1] 필요한 라이브러리 가져오기
────────────────────────────────────
FastAPI 관련 도구 가져오기
        ↓
교과 9·10 JSON을 읽는 함수 가져오기
        ↓
공통 InferenceEvent 규격 가져오기


[2] FastAPI 서버 준비
────────────────────────────────────
FastAPI 프로그램 생성
        ↓
서버 이름과 버전 설정
        ↓
웹 화면 index.html 위치 설정


[3] 공통 검사 결과 불러오기
────────────────────────────────────
load_events() 실행
        ↓
교과 9·10 공통 검사 기록 읽기
        ↓
정상 → 검사 결과 반환
오류 → HTTP 503 반환


[4] 메인 웹 화면 제공
────────────────────────────────────
사용자가 기본 주소(/)에 접속
        ↓
web/index.html 파일 찾기
        ↓
웹브라우저에 HTML 화면 전달


[5] 서버와 결과 파일 상태 확인
────────────────────────────────────
사용자가 /health 요청
        ↓
서버 상태 응답
        ↓
교과 9 JSON 파일 존재 확인
        ↓
교과 10 JSON 파일 존재 확인
        ↓
data_ready: true 또는 false 반환


[6] 검사 결과 목록 조회
────────────────────────────────────
사용자가 검사 결과 요청
        ↓
공통 검사 기록 불러오기
        ↓
조회 조건 확인

  ├─ 전체       → 교과 9·10
  ├─ inspection → 교과 9 OCR
  └─ action     → 교과 10 동작인식

        ↓
요청한 개수만큼 결과 선택
        ↓
JSON 목록 반환


[7] 검사 결과 요약 통계 조회
────────────────────────────────────
사용자가 /api/v1/summary 요청
        ↓
전체 검사 기록 읽기
        ↓
전체 기록 개수 계산
        ↓
교과 9 / 교과 10 기록 개수 계산
        ↓
OK / NG / UNKNOWN 개수 계산
        ↓
요약 결과를 JSON으로 반환


[8] 특정 검사 기록 한 건 조회
────────────────────────────────────
사용자가 event_id를 전달
        ↓
전체 검사 기록에서 ID 검색
        ↓
일치하는 기록이 있는가?
        │
        ├─ 예 → 해당 기록의 상세 JSON 반환
        │
        └─ 아니요 → HTTP 404 오류 반환
```

#### 코드에서 꼭 이해해야 할 핵심

| 구분               | 역할                 | 쉽게 이해하기                 |
| ---------------- | ------------------ | ----------------------- |
| `FastAPI()`      | 웹 서버 애플리케이션 생성     | 안내 창구 만들기               |
| `@app.get()`     | 요청 주소와 함수 연결       | 어떤 요청을 받으면 어떤 일을 할지 지정  |
| `load_events()`  | 공통 JSON 결과 불러오기    | 저장된 검사 기록 가져오기          |
| `return`         | 처리 결과 전달           | 요청한 정보를 사용자에게 돌려주기      |
| `HTTPException`  | 처리할 수 없는 요청의 오류 전달 | 문제가 있는 이유 알려주기          |
| `response_model` | API 응답 데이터 형식 검증   | 전달할 데이터가 정해진 양식에 맞는지 확인 |

### 실습 12. FastAPI 서버를 실행합니다

프로젝트 최상위 폴더에서 실행합니다.

```bash
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
```

터미널에 `Uvicorn running on http://127.0.0.1:8000`이 나타나면 서버가 실행된 것입니다. 서버를 실행한 터미널은 종료하지 않습니다.

웹브라우저에서 다음 주소를 차례대로 열어 확인합니다.

```text
http://127.0.0.1:8000/health
http://127.0.0.1:8000/api/v1/summary
http://127.0.0.1:8000/docs
```

`/health`에서 `data_ready: true`이면 두 JSON 파일이 존재한다는 뜻입니다. `/api/v1/summary`에서는 실제로 읽힌 기록 수를 확인할 수 있습니다.

예시 원본 기록 11건을 사용한 경우 요약 결과는 다음과 같습니다.

```json
{
  "total": 11,
  "by_task": {"inspection": 5, "action": 6},
  "by_decision": {"OK": 7, "NG": 2, "UNKNOWN": 2},
  "mode": "recorded_result"
}
```

자신이 만든 결과물의 건수나 판정 분포가 다르면 숫자는 달라집니다. `/docs`의 **GET /api/v1/events → Try it out → Execute**에서 공통 JSON을 직접 확인해 보세요.

이 단계에서는 아직 `web/index.html`을 작성하지 않았으므로 `/`의 웹 화면은 열리지 않을 수 있습니다. 다음 실습에서 완성합니다.

---

## PART 5. 웹 화면에서 기록을 조회합니다

### 실습 13. HTML 웹 화면을 작성합니다

앞에서는 JSON을 API로 확인했습니다. 이번에는 브라우저에서 제조 검사와 동작인식을 선택하고, 각 기록의 상세 결과를 확인하는 화면을 작성합니다.

**의사코드**

```text
브라우저에서 페이지를 연다
       ↓
FastAPI에 공통 이벤트 목록을 요청한다
       ↓
공통 JSON을 읽는다
       ↓
교과 / 파일명 / 예측 / 판정을 표에 표시한다
       ↓
조회할 교과를 선택한다
       ↓
결과 한 건을 선택하여 상세 JSON을 확인한다
```

**파일: `web/index.html`**

HTML·CSS·JavaScript를 한 파일에 작성합니다.

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>교과 13 · 기존 AI 결과 통합</title>
  <style>
    body {font-family: system-ui, sans-serif; max-width: 1020px; margin: 30px auto; padding: 0 16px; color: #213248;}
    .box {border: 1px solid #cfd7e1; border-radius: 12px; padding: 18px; margin-top: 14px;}
    table {width: 100%; border-collapse: collapse;}
    th, td {border-bottom: 1px solid #e1e5ed; padding: 9px; text-align: left; vertical-align: top;}
    th {background: #f5f7fa;}
    select, button {font: inherit; padding: 7px 10px;}
    button {cursor: pointer;}
    pre {background: #f5f7fa; padding: 14px; overflow-x: auto; white-space: pre-wrap;}
    .ok {color: #14743a; font-weight: bold;}
    .ng {color: #ae2838; font-weight: bold;}
    .unknown {color: #8c6513; font-weight: bold;}
    .note {color: #59677a; font-size: 0.94rem;}
  </style>
</head>
<body>
  <h1>교과 13 · 공통 AI 플랫폼</h1>
  <p><strong>1일차 — 교과 9·10의 실제 저장 결과 조회</strong></p>
  <p class="note">이 화면은 과거에 실행해 저장한 검사·동작인식 결과를 재사용합니다.
    지금 카메라나 AI 모델을 새로 실행하는 실시간 추론 화면은 아닙니다.</p>
  <div class="box">
    <label for="task">결과 종류:</label>
    <select id="task">
      <option value="">전체</option>
      <option value="inspection">교과 9 · OCR 검사</option>
      <option value="action">교과 10 · 동작인식</option>
    </select>
    <button id="load">기록 불러오기</button>
    <p id="summary">결과를 불러오는 중입니다.</p>
    <p id="error" role="alert" class="ng"></p>
  </div>
  <div class="box">
    <h2>저장된 AI 결과</h2>
    <table>
      <thead><tr><th>교과</th><th>입력</th><th>예측</th><th>최종 판정</th><th>상세</th></tr></thead>
      <tbody id="events"></tbody>
    </table>
  </div>
  <div class="box">
    <h2>선택한 결과의 공통 JSON</h2>
    <pre id="detail">표에서 'JSON 보기'를 누르세요.</pre>
  </div>
  <script>
    const task = document.getElementById("task");
    const body = document.getElementById("events");
    const detail = document.getElementById("detail");
    const summary = document.getElementById("summary");
    const error = document.getElementById("error");

    async function loadEvents() {
      error.textContent = "";
      body.replaceChildren();
      detail.textContent = "표에서 'JSON 보기'를 누르세요.";
      try {
        const query = task.value ? `?task=${encodeURIComponent(task.value)}` : "";
        const [response, summaryResponse] = await Promise.all([
          fetch(`/api/v1/events${query}`), fetch("/api/v1/summary")
        ]);
        if (!response.ok || !summaryResponse.ok) throw new Error("API 응답을 확인하세요.");
        const events = await response.json();
        const counts = await summaryResponse.json();
        summary.textContent = `전체 ${counts.total}건 · OK ${counts.by_decision.OK}건 · ` +
          `NG ${counts.by_decision.NG}건 · UNKNOWN ${counts.by_decision.UNKNOWN}건`;
        for (const event of events) {
          const tr = document.createElement("tr");
          for (const value of [event.task, event.input_ref, event.predicted_label ?? "미기록", event.decision]) {
            const td = document.createElement("td");
            td.textContent = value;
            if (value === event.decision) td.className = value.toLowerCase();
            tr.append(td);
          }
          const cell = document.createElement("td");
          const button = document.createElement("button");
          button.textContent = "JSON 보기";
          button.addEventListener("click", () => {
            detail.textContent = JSON.stringify(event, null, 2);
          });
          cell.append(button);
          tr.append(cell);
          body.append(tr);
        }
      } catch (ex) {
        error.textContent = `데이터를 불러오지 못했습니다: ${ex.message}`;
      }
    }
    document.getElementById("load").addEventListener("click", loadEvents);
    loadEvents();
  </script>
</body>
</html>
```

#### 코드 이해하기

`fetch()`는 FastAPI의 `/api/v1/events`와 `/api/v1/summary`에 요청을 보냅니다. `response.json()`은 응답을 JavaScript에서 사용할 수 있는 객체로 읽습니다.

`task` 선택 값을 변경하면 해당 교과의 결과만 표에 나타납니다. `JSON 보기`를 누르면 선택한 기록의 공통 이벤트가 출력됩니다. 오늘은 **기존 기록만 조회하므로 영상·Bounding Box·Skeleton은 표시되지 않습니다.**

### 실습 14. 웹화면과 FastAPI의 검사 결과를 확인합니다.

앞에서 교과 9의 OCR 검사 결과와 교과 10의 동작인식 결과를 공통 JSON 형식으로 변환하고, FastAPI를 통해 조회하는 프로그램을 만들었습니다.

이번 실습에서는 **우리가 만든 FastAPI가 실제로 검사 결과를 올바르게 전달하는지 확인**하겠습니다.

확인하는 방법은 두 가지입니다.

- **웹브라우저:** 검사 결과를 표와 버튼으로 확인합니다.
- **curl:** 터미널에서 FastAPI에 직접 요청하여 JSON 데이터를 확인합니다.

#### 1. 웹브라우저에서 결과를 확인합니다

FastAPI 서버가 실행 중인 상태에서 다음 주소를 엽니다.

```text
http://127.0.0.1:8000/
```

![[Pasted image 20261024201039.png]]

![[Pasted image 20261024201111.png]]

웹 화면에서는 다음과 같이 검사 결과를 선택할 수 있습니다.

| 선택 항목 | 화면에 나타나는 내용 |
|---|---|
| 전체 | 교과 9의 OCR 결과와 교과 10의 동작인식 결과 |
| 교과 9 · OCR 검사 | 제품 이미지에서 읽은 문자와 OK/NG 판정 |
| 교과 10 · 동작인식 | 영상에서 예측한 동작과 기대 동작의 일치 여부 |

표에서 **JSON 보기** 버튼을 누르면 해당 검사 결과의 자세한 내용을 확인할 수 있습니다.

#### 2. curl 테스트란 무엇인가요?

`curl`은 **터미널에서 웹 서버에 요청을 보내고 응답을 확인하는 도구**입니다.

평소에는 웹사이트를 확인할 때 브라우저에서 주소를 입력합니다. `curl`은 같은 일을 터미널의 
명령어로 수행합니다.

쉽게 비유하면 다음과 같습니다.

- **FastAPI:** 검사 결과를 보관하고 요청받은 정보를 전달하는 안내 직원
- **웹브라우저:** 안내 직원에게 결과를 요청하고 보기 좋게 화면에 보여주는 창구
- **curl:** 안내 직원에게 직접 질문하여 받은 결과를 터미널에 출력하는 도구

따라서 `curl`을 이용하면 웹 화면을 거치지 않고도 **FastAPI가 어떤 JSON 데이터를 전달하는지 직접 확인**할 수 있습니다.

#### 3. curl 명령어를 실행합니다

FastAPI 서버를 실행한 터미널은 그대로 두고, 새로운 WSL 터미널을 엽니다.

**① 교과 9의 OCR 검사 결과를 확인합니다.**

```bash
curl -s 'http://127.0.0.1:8000/api/v1/events?task=inspection' | python3 -m json.tool
```

**② 교과 10의 동작인식 결과를 확인합니다.**

```bash
curl -s 'http://127.0.0.1:8000/api/v1/events?task=action' | python3 -m json.tool
```

**명령어를 이해해 봅시다.**

| 명령어 | 의미 |
|---|---|
| `curl` | FastAPI에 HTTP 요청을 보냅니다. |
| `-s` | 요청 진행 표시와 일반 오류 메시지를 숨겨 출력을 간단하게 합니다. |
| `127.0.0.1:8000` | 현재 컴퓨터에서 실행 중인 FastAPI 서버 주소입니다. |
| `/api/v1/events` | 저장된 검사 결과를 조회하는 API 주소입니다. |
| `?task=inspection` | 교과 9의 OCR 결과만 요청합니다. |
| `?task=action` | 교과 10의 동작인식 결과만 요청합니다. |
| `\| python3 -m json.tool` | 응답받은 JSON을 줄바꿈과 들여쓰기로 읽기 쉽게 출력합니다. |

즉, 두 명령어의 차이는 **어떤 교과의 결과를 조회할 것인지 선택하는 부분**입니다.

#### 4. 출력된 JSON 결과를 이해합니다

교과 10의 동작인식 결과를 조회하면 다음과 같은 데이터가 나타납니다.

아래는 실제 출력 결과 중 주요 항목만 가져온 것입니다.

```json
{
  "event_id": "action:job_df7976099734",
  "task": "action",
  "source_mode": "recorded_result",
  "input_ref": "job_df7976099734.mp4",
  "predicted_label": "bend_return",
  "expected_label": "bend_return",
  "confidence": 0.3133,
  "decision": "OK",
  "reasons": []
}
```

각 항목이 무엇을 의미하는지 살펴보겠습니다.

| JSON 항목 | 쉽게 이해하기 | 실제 결과 |
|---|---|---|
| `event_id` | 검사 결과를 구별하는 고유번호 | `action:job_df7976099734` |
| `task` | 어떤 종류의 검사인가요? | `action` (동작인식) |
| `source_mode` | 새로 분석한 결과인가요, 이전에 저장된 결과인가요? | `recorded_result` (이전 기록) |
| `input_ref` | 어떤 영상을 분석했나요? | `job_df7976099734.mp4` |
| `predicted_label` | AI가 어떤 동작으로 예측했나요? | `bend_return` |
| `expected_label` | 원래 기대했던 동작은 무엇인가요? | `bend_return` |
| `confidence` | 모델이 출력한 예측 신뢰도는 얼마인가요? | `0.3133` |
| `decision` | 예측 동작과 기대 동작이 일치했나요? | `OK` |
| `reasons` | 별도로 기록된 판정 이유가 있나요? | `[]` (없음) |

위 결과에서는 AI가 `bend_return` 동작을 예측했고, 기대했던 동작도 `bend_return`이므로 최종 결과가 `OK`로 기록되었습니다.

단, `confidence`는 모델이 출력한 예측 신뢰도이며, 모델 전체의 정확도를 의미하지는 않습니다.

#### 5. UNKNOWN 결과는 무엇인가요?

모든 영상에서 동작을 정확하게 판정할 수 있는 것은 아닙니다.

실제 출력 결과에는 다음과 같은 기록도 있습니다.

```json
{
  "predicted_label": "NO_PREDICTION",
  "expected_label": "leg_raise_lower",
  "confidence": null,
  "decision": "UNKNOWN",
  "reasons": ["동작을 확정하지 못했습니다."]
}
```

이 결과는 AI가 동작을 확정하지 못했다는 의미입니다.

따라서 `NG`로 판단하지 않고 `UNKNOWN`으로 기록합니다.

여기서 세 가지 판정의 의미를 구분해야 합니다.

| 판정 | 교과 10에서의 의미 |
|---|---|
| `OK` | 예측한 동작과 기대 동작이 일치합니다. |
| `NG` | 확정된 예측 동작이 기대 동작과 다릅니다. |
| `UNKNOWN` | 동작을 확정하지 못했거나 비교 결과가 불확실합니다. |

`UNKNOWN`은 오류가 발생했다는 뜻이 아니라, **현재 결과만으로는 동작을 확정하기 어렵다는 의미**입니다.

또한 `NG`가 반드시 위험행동을 의미하는 것도 아닙니다. 이 수업에서는 기대 동작과 다른지를 비교한 결과입니다.

#### 6. 최종 결과를 확인합니다

실제로 출력된 교과 10의 결과는 총 6건이었습니다.

| 구분 | 결과 |
|---|---:|
| 전체 동작인식 기록 | 6건 |
| OK | 4건 |
| NG | 0건 |
| UNKNOWN | 2건 |

이는 이전 교과 10에서 저장한 기록을 읽어 공통 JSON으로 변환한 결과입니다.

**오늘 새롭게 동작인식 모델을 실행한 것이 아니며, 아직 PostgreSQL 데이터베이스에 저장한 것도 아닙니다.**

### 이번 실습의 핵심 정리

이번 실습에서 확인한 것은 다음 흐름입니다.

```text
교과 9·10에서 저장한 JSON 결과
                ↓
       공통 InferenceEvent 변환
                ↓
           FastAPI 서버
                ↓
         ┌──────┴──────┐
         ↓             ↓
      웹브라우저       curl 테스트
         ↓             ↓
      결과 표 확인     JSON 직접 확인
```

**즉, 웹브라우저와 curl은 같은 FastAPI에 요청하는 서로 다른 방법입니다.**

웹브라우저에서는 사람이 보기 편하게 결과를 확인하고, `curl`에서는 서버가 전달하는 실제 JSON 데이터를 직접 확인합니다.

이 과정이 정상적으로 동작하면 **2일차에는 오늘 확인한 공통 JSON을 PostgreSQL 테이블에 저장하고 조회하는 기능**을 추가합니다.

---

## PART 6. 작성한 코드가 올바르게 동작하는지 자동으로 확인합니다

### 실습 15. 자동 테스트 파일을 작성합니다

#### 1. 자동 테스트란 무엇인가요?

앞에서는 웹브라우저와 `curl`을 사용하여 교과 9·10의 검사 결과가 정상적으로 나타나는지 확인했습니다.

그런데 화면에 결과가 보인다고 해서 **프로그램의 모든 기능이 올바르게 동작한다고 판단할 수는 없습니다.**

예를 들어 검사 결과가 일부 누락되거나, 동작을 예측하지 못한 `UNKNOWN`이 잘못해서 `NG`로 바뀌어도 화면만 보고는 발견하기 어려울 수 있습니다.

이러한 문제를 확인하기 위해 **자동 테스트(Automated Test)**를 사용합니다.

자동 테스트는 사람이 일일이 확인하는 대신, 프로그램이 정해진 검사 항목을 실행하고 결과가 
올바른지 판단하는 방법입니다.

쉽게 비유하면, 제품이 출고되기 전에 검사표를 이용하여 정상 작동 여부를 하나씩 확인하는 것과 같습니다.

#### 2. Pytest란 무엇인가요?

**Pytest는 Python 프로그램이 올바르게 작동하는지 자동으로 검사해 주는 도구**입니다.

우리가 미리 작성한 검사 조건을 실행하고, 예상한 결과와 실제 결과가 일치하면 `PASS(통과)`, 
일치하지 않으면 `FAIL(실패)`로 알려줍니다.

이번 실습에서는 Pytest로 다음 내용을 확인하겠습니다.

|검사 대상|무엇을 확인하나요?|
|---|---|
|검사 기록 개수|교과 9·10의 원본 기록이 누락되지 않고 변환되었나요?|
|JSON 변환|교과 9에 없는 신뢰도를 임의로 만들지 않았나요?|
|판정 결과|`OK`, `NG`, `UNKNOWN`이 올바르게 구분되나요?|
|FastAPI 조회|기록 목록·상세·요약 정보를 정상적으로 가져오나요?|
|잘못된 요청 처리|존재하지 않는 기록이나 잘못된 요청을 적절하게 처리하나요?|
|웹 화면|메인 페이지가 정상적으로 열리나요?|

학생마다 OCR 검사나 동작인식을 실행한 횟수가 다르므로 저장된 결과 개수도 다를 수 있습니다.

이번 테스트에서는 **기존 JSON에 저장된 검사 결과가 공통 JSON으로 변환될 때 빠짐없이 전달되었는지** 확인합니다.

예를 들어 OCR 검사 결과가 5건 저장되어 있다면 변환 후에도 5건이어야 합니다.

또한 동작을 예측하지 못한 경우에는 `NG`가 아닌 `UNKNOWN`으로 올바르게 처리되는지도 확인합니다.

#### 3. 자동 테스트는 어떻게 진행되나요?

```
자동 테스트 파일 실행
          ↓
정해진 검사 항목을 하나씩 실행
          ↓
실제 결과와 예상 결과 비교
          ↓
    ┌─────┴─────┐
    ↓           ↓
  일치        불일치
    ↓           ↓
 PASS(통과)  FAIL(실패)
```

#### 4. 자동 테스트 파일을 작성합니다

**파일:** `**tests/test_platform.py**`

이 파일에는 지금까지 작성한 JSON 변환 함수와 FastAPI가 예상대로 작동하는지 확인하는 테스트 코드를 작성합니다.

이번 테스트는 **이전에 저장한 AI 검사 결과를 올바르게 변환하고 조회하는지 확인하는 것**입니다. OCR이나 동작인식 모델을 새로 실행하여 예측 정확도를 평가하는 테스트는 아닙니다.

**핵심:** 자동 테스트를 작성해 두면 앞으로 프로그램을 수정하거나 새로운 기능을 추가하더라도, 
기존 기능이 정상적으로 유지되고 있는지 빠르게 확인할 수 있습니다.

```python
"""
교과 13 · 1일차 자동 테스트

교과 9 OCR 검사 결과와 교과 10 동작인식 결과가
공통 JSON으로 올바르게 변환되고,
FastAPI에서 정상적으로 조회되는지 검사합니다.
"""

# ============================================================
# [준비] 테스트에 필요한 라이브러리와 함수 가져오기
# ============================================================

import json
from pathlib import Path

# FastAPI 서버를 실제로 실행하지 않고 API를 테스트하는 도구
from fastapi.testclient import TestClient

# 앞에서 작성한 FastAPI 프로그램
from app.main import app

# 앞에서 작성한 JSON 변환 함수와 결과 파일 경로
from app.records import (
    ACTION_FILE,
    INSPECTION_FILE,
    action_to_event,
    inspection_to_event,
    load_events,
)

# FastAPI에 요청을 보내는 테스트용 객체 생성
client = TestClient(app)


# ============================================================
# [1] 서버 상태 확인
# FastAPI가 정상 응답하고 결과 JSON 파일이 준비되었는지 검사
# ============================================================

def test_health():

    # /health API에 요청
    response = client.get("/health")

    # HTTP 200이면 요청에 정상 응답한 것
    assert response.status_code == 200

    # 두 교과의 결과 JSON 파일이 준비되었는지 확인
    assert response.json()["data_ready"] is True


# ============================================================
# [2] 원본 기록과 변환 결과의 개수 비교
# OCR·Action 결과가 누락되지 않고 변환되었는지 검사
# ============================================================

def test_imported_results_match_source_counts():

    # 교과 9에서 가져온 OCR 검사 기록 읽기
    original_inspections = json.loads(
        INSPECTION_FILE.read_text(encoding="utf-8")
    )

    # 교과 10에서 가져온 동작인식 기록 읽기
    original_actions = json.loads(
        ACTION_FILE.read_text(encoding="utf-8")
    )

    # 두 교과의 결과를 공통 InferenceEvent로 변환
    events = load_events()

    # 원본 기록이 두 교과 모두 존재하는지 확인
    assert original_inspections and original_actions

    # OCR 원본 기록 개수와 변환 결과 개수 비교
    assert sum(
        event.task == "inspection" for event in events
    ) == len(original_inspections)

    # Action 원본 기록 개수와 변환 결과 개수 비교
    assert sum(
        event.task == "action" for event in events
    ) == len(original_actions)


# ============================================================
# [3] OCR 신뢰도 확인
# 원본에 없는 신뢰도를 임의로 만들지 않았는지 검사
# ============================================================

def test_inspection_confidence_is_not_invented():

    # FastAPI에서 OCR 결과만 요청
    data = client.get(
        "/api/v1/events?task=inspection"
    ).json()

    # OCR 결과가 존재하고 모든 confidence가 None인지 확인
    assert data and all(
        row["confidence"] is None for row in data
    )


# ============================================================
# [4] OCR의 NG 판정과 이유 확인
# 원본의 불일치 판정과 NG 이유가 유지되는지 검사
# ============================================================

def test_inspection_keeps_original_ng_reason():

    # 테스트용 OCR 결과 1건 준비
    # 실제 원본 데이터를 수정하지 않습니다.
    sample = {
        "request_id": "practice-01",
        "inspected_at": "2026-09-15T18:00:00",
        "file_name": "product.png",
        "inspection_module": "day06_ocr",
        "rule_version": "day06-ocr-v1.0",
        "service_version": "v1",
        "measurements": {
            "ocr_text": "1234567890123",
            "expected_text": "1234567890124",
            "pattern_ok": True,
            "expected_match": False,
        },
        "final_decision": "NG",
        "reasons": ["NG_EXPECTED_MISMATCH"],
    }

    # OCR 결과를 공통 InferenceEvent로 변환
    event = inspection_to_event(sample)

    # NG 판정이 유지되고 신뢰도는 없는지 확인
    assert event.decision == "NG" and event.confidence is None

    # 원본의 NG 이유가 유지되는지 확인
    assert "NG_EXPECTED_MISMATCH" in event.reasons


# ============================================================
# [5] 동작을 예측하지 못한 경우 확인
# NO_PREDICTION은 NG가 아니라 UNKNOWN이어야 함
# ============================================================

def test_action_no_prediction_means_unknown():

    # 동작 예측에 실패한 테스트용 결과
    sample = {
        "source_path": "reports/day10/web/jobs/test/result.json",
        "result": {
            "job_id": "test",
            "created_at": "2026-09-24T15:00:00+09:00",
            "source_file": "demo.mp4",
            "detected_action": "NO_PREDICTION",
            "expected_action": "wave",
            "final_match": False,
            "latest_confidence": None,
            "metrics": {},
            "runtime": {},
        },
    }

    # 동작을 확정하지 못했다면 UNKNOWN인지 확인
    assert action_to_event(sample).decision == "UNKNOWN"


# ============================================================
# [6] 예측 동작과 기대 동작이 다른 경우 확인
# NG는 동작 불일치를 뜻하며 위험행동 판정과는 구별
# ============================================================

def test_action_mismatch_not_equated_with_safety_risk():

    # 예측 동작과 기대 동작이 다른 테스트용 결과
    sample = {
        "source_path": "reports/day10/web/jobs/test/result.json",
        "result": {
            "job_id": "test",
            "created_at": "2026-09-24T15:00:00+09:00",
            "source_file": "demo.mp4",
            "detected_action": "sit_stand",
            "expected_action": "wave",
            "final_match": False,
            "latest_confidence": 0.8,
            "metrics": {},
            "runtime": {},
        },
    }

    # 공통 InferenceEvent로 변환
    event = action_to_event(sample)

    # 서로 다른 동작이므로 NG인지 확인
    assert event.decision == "NG"

    # 판정 이유에 기대 동작과의 불일치가 기록되는지 확인
    assert "기대 동작" in event.reasons[0]


# ============================================================
# [7] 전체 요약 통계 확인
# API에 표시되는 건수와 실제 이벤트 건수가 일치하는지 검사
# ============================================================

def test_summary_matches_events():

    # FastAPI에서 전체 요약 통계 요청
    response = client.get("/api/v1/summary")

    # API가 반환한 요약 결과
    result = response.json()

    # 실제 공통 이벤트 목록
    events = load_events()

    # 전체 기록 개수 비교
    assert result["total"] == len(events)

    # OK / NG / UNKNOWN 각각의 개수 비교
    for decision in ("OK", "NG", "UNKNOWN"):

        assert result["by_decision"][decision] == sum(
            x.decision == decision for x in events
        )


# ============================================================
# [8] 교과별 결과 필터 확인
# Action을 선택하면 동작인식 결과만 반환되는지 검사
# ============================================================

def test_filter_by_task():

    # Action 결과만 조회
    response = client.get("/api/v1/events?task=action")

    # API 요청 성공 여부
    assert response.status_code == 200

    # 모든 결과가 action인지 확인
    assert response.json() and all(
        x["task"] == "action" for x in response.json()
    )


# ============================================================
# [9] 검사 결과 한 건 조회
# event_id로 요청하면 해당 결과를 찾아주는지 검사
# ============================================================

def test_get_event_by_id():

    # 기존 기록에서 첫 번째 event_id 가져오기
    event_id = load_events()[0].event_id

    # 해당 event_id로 상세 결과 요청
    response = client.get(f"/api/v1/events/{event_id}")

    # 정상 응답하고 같은 event_id를 반환하는지 확인
    assert (
        response.status_code == 200
        and response.json()["event_id"] == event_id
    )


# ============================================================
# [10] 존재하지 않는 기록 요청
# 없는 event_id를 요청하면 HTTP 404를 반환하는지 검사
# ============================================================

def test_missing_event_is_404():

    # 실제로 존재하지 않는 ID 요청
    response = client.get(
        "/api/v1/events/not-a-real-id"
    )

    # HTTP 404 = 요청한 기록을 찾을 수 없음
    assert response.status_code == 404


# ============================================================
# [11] 잘못된 검색 조건 확인
# 허용하지 않은 task 값이면 HTTP 422를 반환하는지 검사
# ============================================================

def test_invalid_task_is_422():

    # inspection/action이 아닌 잘못된 값 전달
    response = client.get(
        "/api/v1/events?task=wrong"
    )

    # HTTP 422 = 요청한 데이터의 값이 올바르지 않음
    assert response.status_code == 422


# ============================================================
# [12] 웹 화면 정상 응답 확인
# index.html이 정상적으로 전달되는지 검사
# ============================================================

def test_web_page():

    # 메인 웹 화면 요청
    result = client.get("/")

    # 정상 응답하고 페이지에 "교과 13" 문자가 있는지 검사
    assert (
        result.status_code == 200
        and "교과 13" in result.text
    )
```

#### 의사코드

```
[준비] 자동 테스트 환경 준비
──────────────────────────────────────
JSON 처리 도구 가져오기
FastAPI 테스트 도구 가져오기
기존 JSON 변환 함수 가져오기
테스트용 FastAPI 객체 생성


[1] 서버 상태 확인
──────────────────────────────────────
/health API 요청
       ↓
HTTP 200인지 확인
       ↓
결과 JSON 파일이 준비되었는지 확인


[2] 원본 기록과 변환 결과 개수 비교
──────────────────────────────────────
교과 9 OCR 결과 JSON 읽기
교과 10 Action 결과 JSON 읽기
       ↓
두 결과를 공통 이벤트로 변환
       ↓
OCR 원본 건수 = 변환된 OCR 건수?
Action 원본 건수 = 변환된 Action 건수?
       ↓
모두 일치하면 통과


[3] OCR 신뢰도 확인
──────────────────────────────────────
교과 9 OCR 기록 조회
       ↓
confidence 값 확인
       ↓
모든 기록이 None이면 통과
(원본에 없는 신뢰도를 만들지 않음)


[4] OCR의 NG 판정과 이유 확인
──────────────────────────────────────
서로 다른 SN 문자열을 가진
테스트용 OCR 결과 준비
       ↓
공통 이벤트로 변환
       ↓
NG 판정 유지 확인
NG 이유 유지 확인
신뢰도 None 확인


[5] 동작 예측 실패 확인
──────────────────────────────────────
NO_PREDICTION 결과 준비
       ↓
공통 이벤트로 변환
       ↓
decision이 UNKNOWN이면 통과


[6] 예측 동작 불일치 확인
──────────────────────────────────────
예측 동작: sit_stand
기대 동작: wave
       ↓
공통 이벤트로 변환
       ↓
decision이 NG인지 확인
       ↓
판정 이유가 동작 불일치인지 확인


[7] 요약 통계 확인
──────────────────────────────────────
/api/v1/summary 요청
       ↓
전체 기록 건수 비교
       ↓
OK 건수 비교
NG 건수 비교
UNKNOWN 건수 비교
       ↓
모든 개수가 일치하면 통과


[8] 교과별 검색 확인
──────────────────────────────────────
task=action으로 API 요청
       ↓
요청 성공 여부 확인
       ↓
동작인식 결과만 반환되면 통과


[9] 개별 기록 조회 확인
──────────────────────────────────────
기존 event_id 1개 선택
       ↓
해당 ID로 상세 API 요청
       ↓
같은 event_id의 결과가 반환되면 통과


[10] 존재하지 않는 기록 요청
──────────────────────────────────────
없는 event_id로 API 요청
       ↓
HTTP 404가 반환되면 통과


[11] 잘못된 검색 조건 요청
──────────────────────────────────────
task=wrong으로 API 요청
       ↓
HTTP 422가 반환되면 통과


[12] 웹 화면 확인
──────────────────────────────────────
메인 페이지 / 요청
       ↓
HTTP 200인지 확인
       ↓
페이지에 "교과 13"이 있으면 통과
```

#### 자동 테스트 실행

프로젝트 최상위에서 실행합니다.

```bash
python -m pytest -q
```

테스트 결과는 12개 테스트가 모두 통과했으므로 기존 AI 결과를 공통 JSON으로 변환하고 FastAPI에서 조회하는 코드가 정상적으로 동작함을 확인했습니다. 경고 1개는 테스트 라이브러리 관련 안내이므로 현재 실습 진행에는 문제가 없습니다.

결과는 아래와 같습니다.

```text
............                            [100%]

12 passed, 1 warning in 1.20s
```

|출력 내용|의미|결과|
|---|---|---|
|`............`|12개 테스트 각각 통과|정상|
|`[100%]`|전체 테스트 실행 완료|정상|
|`12 passed`|12개 테스트 모두 성공|정상|
|`1 warning`|주의할 사항 1개 발생|실행에는 영향 없음|
|`1.20s`|전체 테스트 실행시간|1.20초|
#### 실제로 검증한 내용은?

1일차에 작성한 `tests/test_platform.py`를 기준으로 다음 기능을 확인한 것입니다.

|검증 대상|확인한 내용|
|---|---|
|데이터 가져오기|교과 9·10의 기록 개수가 변환 후에도 일치하는가?|
|공통 JSON 변환|OCR에 없는 신뢰도를 임의로 생성하지 않는가?|
|OK/NG/UNKNOWN|OCR 판정 이유와 동작인식 판정 규칙이 유지되는가?|
|FastAPI|결과 목록·상세·요약 API가 정상 응답하는가?|
|예외 처리|잘못된 요청이나 존재하지 않는 기록을 올바르게 처리하는가?|
|웹 화면|메인 페이지가 정상적으로 반환되는가?|

즉, 교과 9·10의 기존 결과 JSON을 공통 구조로 변환하고, FastAPI에서 조회하는 기능이 자동 테스트를 통과한 것입니다.

단, 실제 OCR 또는 동작인식 모델을 다시 실행한 테스트는 아닙니다. PostgreSQL 저장 기능도 아직 검증한 것은 아닙니다.

## 3. 경고 1개는 무엇인가요?

```
StarletteDeprecationWarning:
Using `httpx` with `starlette.testclient`
is deprecated; install `httpx2` instead.
```

현재 FastAPI의 테스트 도구가 내부적으로 사용하는 라이브러리와 관련된 기능 변경 예고 경고입니다.

- `Error`가 아니므로 테스트가 중단되지 않았습니다.
    
- 작성한 JSON 변환 코드에 오류가 있다는 뜻은 아닙니다.
    
- 현재 테스트가 모두 통과하므로 당장 코드를 수정할 필요는 없습니다.
    
- 추후 라이브러리 업데이트 시 호환성을 확인하면 됩니다.
    

---

## PART 7. 오늘 만든 프로젝트를 정리합니다

### 실습 17. 시스템 연결 구조를 문서로 남깁니다

오늘 만든 플랫폼은 2일차부터 계속 확장합니다. 현재 연결된 기능과 앞으로 추가할 기능을 구분해서 기록합니다.

**파일: `docs/architecture.md`**

```markdown
# 교과 13 공통 AI 플랫폼 구조

## 1일차에 완성한 흐름

교과 9 OCR 저장 기록 ───┐
                        ├→ 공통 InferenceEvent → FastAPI → 웹 UI
교과 10 Action 저장 기록 ┘

- 결과 인수: `scripts/01_import_results.py`
- 인수 데이터: `handoff/records/`
- 공통 규격: `app/schemas.py`
- 결과 변환: `app/records.py`
- API: `app/main.py`
- 화면: `web/index.html`
- 테스트: `tests/test_platform.py`

## 다음 일차에 추가할 기능

- 2일차: PostgreSQL · SQLAlchemy · CRUD
- 3일차: 영상·Keypoint 데이터 연결 · Redis · Celery
- 4일차: 교과 9·10 실제 추론 · 실시간 웹 화면
- 5일차: Edge 모델 학습 · ONNX
- 6일차: Raspberry Pi CPU 추론 · GPIO
- 7일차: Docker · Docker Compose
- 8~9일차: Jetson GPU · TensorRT · 실시간 최적화
- 10일차: Prometheus · Grafana
- 11일차: GPU Cloud 배포
- 12일차: Git CI/CD · Rollback
- 13일차: 최종 검증 및 프로젝트 인계

1일차는 기존에 저장된 AI 결과를 조회합니다. 실제 모델 재추론은 아직 연결하지 않았습니다.
```

### 실습 18. 이전 교과의 어떤 결과를 사용했는지 기록합니다

새 프로젝트에서 사용하는 JSON이 어디에서 왔는지 나중에도 알 수 있도록 인수 내역을 남깁니다.

**파일: `handoff/README.md`**

```markdown
# 교과 9·10 결과 인수 내역

## 교과 9

- 사용 파일: `reports/day10/service_history.json`
- 내용: OCR 검사 이력, 이미지명, OCR Text, Expected Text, OK/NG, Reasons
- 공통 JSON 변환: `inspection_to_event()`
- OCR Confidence는 원본에 없으므로 `null`

## 교과 10

- 사용 파일: `reports/day10/web/jobs/*/result.json`
- 내용: Action 예측 결과, 기대 Action, 신뢰도, 최종 비교
- 공통 JSON 변환: `action_to_event()`
- UNKNOWN / NO_PREDICTION은 판정 유보

## 결과 인수 방법

기존 교과 프로젝트에 결과가 있으면 그 위치에서 읽습니다.
기존 결과가 없으면 `handoff/provided/`에 배치한 자료를 사용합니다.

실행 명령: `python scripts/01_import_results.py`

## 다음 수업 연결

2일차: 오늘의 공통 이벤트를 PostgreSQL에 저장합니다.
4일차: 실제 AI 모델과 카메라·영상 입력을 연결합니다.

## 보관 주의

원본 결과 파일, 대체 자료, 비공개 이미지·영상·모델은 허가 없이 공개하지 않습니다.
```

### 실습 19. README에 실행 방법을 기록합니다

프로젝트를 다시 실행할 때 필요한 명령을 남깁니다.

**파일: `README.md`**

```markdown
# 교과 13 · AI 서비스 플랫폼

## 1일차 완성 기능

교과 9 OCR 검사 기록과 교과 10 Action Recognition 기록을
공통 `InferenceEvent`로 변환하고 FastAPI와 웹 화면으로 조회합니다.

현재는 과거에 저장된 기록을 사용하는 방식입니다.
실시간 AI 모델·PostgreSQL·Docker는 아직 연결하지 않았습니다.

## 실행 환경

- Python 3.10 이상 (Ubuntu/WSL)
- `requirements.txt`의 패키지

## 실행 순서

1. 기존 교과 9·10 결과 파일을 확인합니다.
2. 결과가 없으면 `handoff/provided/`에 제공된 기록을 배치합니다.
3. 다음 명령을 실행합니다.

    python3 -m venv .venv
    source .venv/bin/activate
    python -m pip install -r requirements.txt
    python scripts/01_import_results.py
    python -m pytest -q
    python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000

## 웹 주소

- 웹: http://127.0.0.1:8000/
- API 문서: http://127.0.0.1:8000/docs
- 기록 목록: http://127.0.0.1:8000/api/v1/events
- 요약: http://127.0.0.1:8000/api/v1/summary

## 다음 일차

2일차에는 같은 프로젝트에서 ERD·PostgreSQL·SQLAlchemy·CRUD를 만듭니다.
```

### 실습 20. Git에 올리지 않을 파일을 지정합니다

AI 모델·영상·기존 교과의 원본 기록은 오늘 새로 작성한 **소스코드와 구분하여 보관**합니다. `.gitignore`를 작성하면 지정한 파일과 폴더가 실수로 새 Git 기록에 포함되는 것을 줄일 수 있습니다.

**파일: `.gitignore`**

```gitignore
.venv/
__pycache__/
.pytest_cache/
*.py[cod]
.env
.env.*
!.env.example
handoff/records/
handoff/provided/
*.pt
*.pth
*.joblib
*.onnx
*.engine
*.mp4
*.avi
```

`handoff/records/`와 `handoff/provided/`는 Git 업로드 대상에서 제외됩니다. **이미 Git이 추적하고 있는 파일은 `.gitignore`만 추가해도 자동으로 제외되지 않습니다.** 따라서 아래에서 `git status`를 반드시 확인합니다.

---

# 6. 오늘 작성한 내용의 최종 정리 및 GitHub 업로드

## 6.1. 오늘의 최종 결과

오늘은 교과 9의 OCR 검사와 교과 10의 Action Recognition에서 **이미 저장해 둔 실제 결과**를 가져왔습니다. 서로 다른 JSON의 필드를 `InferenceEvent`라는 공통 구조로 변환하고, FastAPI·웹브라우저에서 함께 조회하도록 연결했습니다.

기존 데이터에 없는 값을 임의로 만들지 않았으며, 동작 예측이 확정되지 않은 경우는 `UNKNOWN`으로 보존했습니다. 이 결과는 **2일차에 PostgreSQL 테이블을 만들 때 사용할 기본 이벤트 데이터**가 됩니다.

```text
교과 9 OCR 결과      교과 10 Action 결과
       └──────────────┬──────────────┘
                      ↓
              공통 InferenceEvent
                      ↓
                    FastAPI
                      ↓
                웹 기록 조회 화면
                      ↓
                 자동 테스트 완료
                      ↓
           2일차 PostgreSQL로 확장
```

## 6.2. 최종 자가 체크리스트

- [ ] 교과 9·10에서 만든 결과 JSON을 확인했거나 제공된 결과 파일을 배치했습니다.
- [ ] `subject13_ai_platform` 폴더와 `.venv`를 생성했습니다.
- [ ] `01_import_results.py`로 두 교과의 결과가 `handoff/records/`에 저장되었습니다.
- [ ] `InferenceEvent`로 JSON을 변환하고 실제 이벤트 건수를 확인했습니다.
- [ ] `/health`, `/api/v1/events`, `/api/v1/summary`, `/docs`를 확인했습니다.
- [ ] 웹 화면에서 두 교과 기록을 선택하고 상세 JSON을 확인했습니다.
- [ ] `python -m pytest -q`에서 **12개 테스트가 통과**했습니다.
- [ ] `README.md`, `docs/architecture.md`, `handoff/README.md`를 작성했습니다.
- [ ] 가상환경과 원본 데이터가 Git 업로드 대상에서 제외되었습니다.

## 6.3. Git에 1일차 작업을 기록합니다

**교과 13의 커밋 메시지 규칙은 `교과13 N일차: 핵심 구현 내용`으로 통일**합니다. 이후 수업도 같은 저장소에 새로운 기능을 추가하고 일차별 커밋을 남깁니다.

프로젝트 최상위에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git init
git status
```

오늘 작성한 소스코드와 문서만 스테이징합니다.

```bash
git add .gitignore requirements.txt app scripts web tests docs README.md handoff/README.md
git status
```

`git status`에서 `handoff/records/`, `handoff/provided/`, `.venv/`가 업로드할 파일 목록에 없는지 확인합니다. 이제 커밋합니다.

```bash
git commit -m "교과13 1일차: 기존 AI 결과 통합 및 공통 JSON 조회 플랫폼 구축"
git log -1 --oneline
git status
```

## 6.4. 원격 저장소에 올립니다

외부 GitHub 업로드가 허용된 환경에서는 GitHub에 **빈 저장소**를 만들고 아래 명령을 실행합니다. `YOUR_ID`는 본인의 계정명으로 바꿉니다. 외부 GitHub 사용이 제한된 환경이라면 **로컬 Git 또는 허용된 내부 GitLab/Gitea 저장소**를 사용합니다.

```bash
git branch -M main
git remote add origin https://github.com/YOUR_ID/subject13_ai_platform.git
git push -u origin main
```

이미 `origin`이 존재한다면 `git remote -v`로 확인한 후, 필요할 때만 URL을 변경합니다. 모델 파일, 원본 데이터, 비밀번호, 인증 토큰은 원격 저장소에 올리지 않습니다.
