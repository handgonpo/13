## 1. 오늘의 목표

지금까지 만든 플랫폼은 이미 계산하여 보관한 교과 9·10의 결과를 읽거나, PostgreSQL에 저장하고, 데이터 파일로 내보낼 수 있습니다. 그러나 **새로 입력한 이미지나 카메라 영상을 AI가 실제 분석하는 기능은 아직 연결하지 않았습니다.**

4일차에는 두 가지 실제 실행 흐름을 만들겠습니다. 교과 9에서는 준비된 제품 이미지로 **Tesseract OCR 검사를 다시 실행**하고, 교과 10에서는 **IP Webcam의 연속 영상을 YOLO11n-Pose → 38-D Frame Feature → Sliding Window → Random Forest Action Recognition**으로 처리합니다.

오늘 수업을 끝내면 다음 작업을 할 수 있습니다.

- 교과 9 OCR 서비스와 교과 10 Pose/Action 기능을 이전 프로젝트에서 다시 실행합니다.
- 기존 프로젝트마다 다른 Python 가상환경을 유지하면서 HTTP API로 교과 13에 연결합니다.
- 실제 OCR 검사 결과를 공통 `InferenceEvent`로 변환하여 PostgreSQL에 저장합니다.
- 스마트폰 IP Webcam의 영상에서 BBox·Skeleton·Raw/Stable Action·Confidence·FPS를 웹 화면에서 확인합니다.
- 선택한 기준 Action과 안정화된 예측 Action을 비교한 결과를 PostgreSQL에 저장합니다.
- 1~3일차 기록 및 API를 유지하고, 4일차 새 결과만 구분해서 조회합니다.
- API와 데이터 저장을 자동 테스트하고 README·Git으로 결과를 정리합니다.

**오늘의 완성 기준:** `http://127.0.0.1:8000/live` 한 화면에서 실제 OCR 검사를 실행하고, IP Webcam Pose/Action 결과를 확인하여, 각각의 새 기록이 PostgreSQL에 `source_mode=live_inference`로 저장되면 됩니다. 실제 Action 모델의 정답률은 오늘의 연결 확인만으로 보장하지 않습니다.

---

## 2. 오늘 수업의 전체 흐름

![[Pasted image 20261018130112.png]]

```text
[교과 9 실제 OCR 서비스]                       [교과 10 실제 Pose·Action]
  제품 이미지 6장                              스마트폰 IP Webcam
        ↓                                              ↓
  day06 Frozen Rule                             연속 영상 Frame
        ↓                                              ↓
  Tesseract OCR + Regex                      YOLO11n-Pose + COCO17
        ↓                                              ↓
  실제 OK/NG + Reasons                        38-D Frame Feature
        ↓                                              ↓
  기존 FastAPI :8009                         Sliding Window + RF
        ↓                                              ↓
  교과 13 OCR 연동 API                        Stable Action / FPS
        ↓                                              ↓
        │                                     Bridge FastAPI :8010
        │                                              ↓
        └────────────────────┬─────────────────────────┘
                             ↓
                 교과 13 공통 FastAPI :8000
                             ↓
                  공통 InferenceEvent 변환
                             ↓
                PostgreSQL inference_events
                             ↓
              /live 웹 화면 + 저장 기록 조회
                             ↓
                  Pytest · README · Git
                             ↓
          [5일차] TinyCNN / ONNX Edge 모델 준비
```

오늘은 **세 개의 서버가 동시에 실행**됩니다. 교과 9의 OCR 서비스(`8009`), 교과 10 모델을 실행하는 Bridge(`8010`), 그리고 1~3일차부터 사용한 교과 13 플랫폼(`8000`)입니다. 서버는 역할이 다르므로 각각의 터미널을 유지합니다.

---

## 3. 실습 전에 알아야 할 핵심

### 3.1. 1~3일차 프로젝트를 그대로 확장합니다

오늘도 `subject13_ai_platform` 하나를 이어서 사용합니다. 이전에 만든 다음 기능을 삭제하거나 다시 작성하지 않습니다.

| 이전 일차 | 그대로 사용할 결과 | 오늘 활용 방법 |
|---|---|---|
| 1일차 | `app/schemas.py`, `app/records.py`, `app/main.py`, `web/index.html` | 기존 OCR 기록 변환 함수와 공통 JSON 규격 재사용 |
| 2일차 | `app/db.py`, `app/models.py`, `app/repository.py`, `app/db_api.py` | 같은 PostgreSQL 테이블에 새 실제 추론 이벤트 저장 |
| 3일차 | `app/data_pipeline.py`, `app/pipeline_api.py`, `app/worker_tasks.py`, `web/pipeline.html` | 3일차 비동기 작업 기능 그대로 보존 |

1일차 저장 기록의 `source_mode`는 `recorded_result`, 2일차에서 직접 입력한 CRUD 연습 기록은 `manual_entry`였습니다. 오늘 실제 AI를 다시 실행해서 저장한 데이터에는 **`live_inference`**를 사용합니다. 세 종류를 구분해야 기존 기록과 새 AI 결과가 섞여도 의미를 정확히 알 수 있습니다.

### 3.2. 교과 9·10에서 필요한 기존 작업 결과

교과 9·10에서 직접 작성한 프로젝트가 남아 있다면 전체 폴더를 다시 복사하거나 GitHub ZIP을 다운로드하지 않습니다. 먼저 다음 파일이 있는지만 확인합니다.

| 구분 | 기존 프로젝트의 필수 자료 | 필요한 이유 |
|---|---|---|
| 교과 9 · OCR | `day10_service/service_core.py`, `day10_service/api_main.py` | 이전에 만든 OCR 추론과 FastAPI 재사용 |
| 교과 9 · OCR | `configs/day06_ocr_rules.json`, `configs/day06_labels.csv`, `configs/day10/service_config.json` | 고정된 ROI·PSM·Expected SN·서비스 설정 |
| 교과 9 · OCR | `samples/day06/product/`의 실제 검사 이미지 6장 | 새 OCR 추론 입력. **기록 JSON만으로는 재검사할 수 없습니다.** |
| 교과 10 · Action | `src/common.py`, `src/feature_utils.py`, `src/sequence_utils.py`, `src/realtime_utils.py` | 6·7일차의 특징 계산·Sliding Window를 그대로 재사용 |
| 교과 10 · Action | `configs/project.json`, `configs/day07_realtime.json` | YOLO 장치·이미지 크기·Window·Threshold·Stabilization |
| 교과 10 · Action | `artifacts/day06/rf_action_baseline.joblib` | 학습 완료된 6개 Action Random Forest |
| 교과 10 · Action | `artifacts/day06/rf_feature_columns.json`, `rf_training_info.json` | 학습 당시 Feature 순서·Sequence Step 확인 |
| 교과 10 · Action | `yolo11n-pose.pt` | 17개 관절을 예측하는 실제 YOLO Pose 가중치 |
| 실시간 입력 | IP Webcam 앱을 실행할 스마트폰 | `/video` Stream을 통한 실시간 입력 |

**앞 교과의 결과물이 없는 경우:** 이전 수업에서 결석했거나 모델·원본 이미지가 누락되었다면 위 표의 해당 파일과 폴더를 제공받아 **교과 9·10의 원래 상대경로 그대로** 배치합니다. 오늘 새로운 Action 모델을 훈련하거나 OCR Rule을 다시 만드는 과정은 포함하지 않습니다. 특히 `joblib` 모델은 신뢰할 수 있는 수업 산출물만 사용합니다. 미완성 파일을 임의의 데모 값으로 대체하지 않습니다.

**저장 MP4로 먼저 연결을 확인하고 싶다면:** 교과 10의 `data/day05/videos/S02/S02_A003_T01.mp4`처럼 Sliding Window를 채울 만큼 충분히 긴 영상을 사용할 수 있습니다. 이 경우에도 **실제 YOLO와 Random Forest 추론**을 실행합니다. IP Webcam이 정상 연결되면 동일한 코드에 입력 URL만 바꾸어 진행합니다.

### 3.3. 이미 저장된 결과 조회와 실제 추론을 구분합니다

```text
1~3일차 : 과거 JSON → 공통 이벤트 → PostgreSQL/Export
4일차 OCR : 현재 선택한 이미지 → 실제 Tesseract OCR → 공통 이벤트 → PostgreSQL
4일차 Action : 지금 들어오는 Frame → 실제 Pose·RF → 공통 이벤트 → PostgreSQL
```

오늘 교과 9의 OCR 대상은 **미리 준비된 제품 이미지**입니다. 교과 9 OCR을 IP Webcam 실시간 검사로 바꾸지 않습니다. 스마트폰 IP Webcam은 **교과 10의 Pose·Action 실습**에서 사용합니다.

### 3.4. OCR의 NG와 Action의 NG는 같은 뜻이 아닙니다

교과 9는 **OCR이 읽은 SN과 Expected SN**을 비교합니다. 예를 들어 13자리 숫자 형식은 맞지만 Expected SN이 다르면 `NG_EXPECTED_MISMATCH`입니다. OCR 결과에는 별도의 예측 확률이 없으므로 `confidence=null`을 유지합니다.

교과 10은 **안정화된 Action 예측과 사용자가 선택한 기준 Action**을 비교합니다. 이때 `NG`는 *기준과 다른 동작*이라는 뜻이지 산업안전 사고나 위험행동을 확정했다는 뜻이 아닙니다. `UNKNOWN`은 Window의 품질이나 예측 신뢰도가 충분하지 않아 판정을 유보한 상태입니다. `WARMING_UP`은 아직 Window가 채워지지 않은 준비 단계로, 저장 가능한 판정으로 취급하지 않습니다.

또한 Random Forest의 `raw_confidence`는 **현재 Raw 클래스의 분류 확률**입니다. 여러 예측을 투표하여 만든 `stable_label`의 확률은 아닙니다. 두 라벨이 서로 다르면 공통 이벤트의 `confidence`는 임의로 계산하지 않고 `null`로 보존합니다.

### 3.5. 왜 가상환경과 서버를 분리하나요?

교과 9는 OpenCV·Tesseract, 교과 10은 PyTorch·Ultralytics·scikit-learn, 교과 13은 PostgreSQL·FastAPI·Redis·Celery를 사용합니다. 각 교과의 `.venv`에 이미 설치된 패키지를 활용하고, **모델 가중치나 거대한 라이브러리를 교과 13 가상환경으로 무조건 복사하지 않습니다.**

교과 10의 모델은 같은 컴퓨터에서 `8010` 포트의 별도 Bridge로 실행하지만, 새 Bridge 소스코드 자체는 **교과 13 프로젝트 안의 `bridges/` 폴더에 작성**합니다. 따라서 4일차에 작성한 코드도 교과 13 Git 저장소에서 일관되게 관리할 수 있습니다.

### 3.6. 실시간 처리와 Redis·Celery의 역할을 구분합니다

4일차의 비디오 Frame → Pose → Action 계산은 교과 10의 런타임에서 직접 수행합니다. **Redis·Celery가 영상 추론 FPS를 자동으로 높여 주는 것은 아닙니다.** 오늘 사용자가 검사 버튼이나 저장 버튼을 누르면 FastAPI가 결과를 PostgreSQL에 동기적으로 저장하여 동작을 확인합니다. 3일차의 Celery Export API는 그대로 존재하지만, 오늘 실시간 추론 요청 처리에 끼워 넣지 않습니다.

---

## 4. 오늘 사용할 기술 스택

| 기술 | 오늘 직접 사용하는 역할 |
|---|---|
| Python / 기존 `.venv` | 교과 9·10·13의 분리된 실행환경 |
| OpenCV · Tesseract · pytesseract | 교과 9 제품 이미지의 실제 13자리 SN OCR 검사 |
| YOLO11n-Pose · COCO17 | 교과 10 IP Webcam 영상의 사람 Bounding Box·Skeleton 검출 |
| NumPy · 38-D Feature · Sliding Window | 기존 동작인식 전처리·시간 Sequence 생성 |
| scikit-learn · `joblib` | 교과 10의 이미 학습된 Random Forest 6개 Action 예측 |
| HTTP · HTTPX · FastAPI | 모델별 기존 서비스와 교과 13 플랫폼 연결 |
| Pydantic · SQLAlchemy · PostgreSQL | 공통 JSON 검증과 실제 결과 영구 저장 |
| HTML · JavaScript `fetch()` | 한 페이지에서 OCR 결과·Pose 영상·DB 기록 보기 |
| Pytest · Git | 연결 기능 검증 및 버전 기록 |

---

# 5. 처음부터 끝까지 따라 하는 상세 실습

## PART 1. 3일차 프로젝트와 준비물을 확인합니다

### 실습 1. 교과 13 프로젝트와 기존 가상환경을 다시 엽니다

새 프로젝트를 만들지 않습니다. WSL Ubuntu 터미널에서 다음을 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
pwd
which python
ls app/main.py app/db.py app/models.py app/pipeline_api.py web/pipeline.html
```

`pwd`의 끝이 `subject13_ai_platform`이고, `which python`에 `.venv/bin/python`이 표시되면 정상입니다. `app/pipeline_api.py`까지 보인다면 1~3일차 코드가 이어진 상태입니다.

PostgreSQL이 실행 중인지 확인합니다.

```bash
pg_isready -h 127.0.0.1 -p 5432
python -m scripts.03_import_records_to_db
```

DB 이관 스크립트를 다시 실행하더라도 `event_id`가 같은 기존 기록을 중복 저장하지 않는 것이 2일차의 구현입니다. 오늘은 이 PostgreSQL을 새로 설치하거나 초기화하지 않습니다.

### 실습 2. 교과 9·10의 원본 프로그램과 모델 파일을 확인합니다

WSL 터미널에서 아래 명령을 실행합니다. 교과 9·10 프로젝트가 다른 위치에 있다면 경로 앞부분을 본인 경로에 맞게 바꿉니다.

```bash
ls -lh ~/ai_vision/subject9_machinevision/day10_service/api_main.py
ls -lh ~/ai_vision/subject9_machinevision/configs/day06_ocr_rules.json
ls -lh ~/ai_vision/subject9_machinevision/configs/day06_labels.csv
find ~/ai_vision/subject9_machinevision/samples/day06/product \
  -maxdepth 1 -type f \( -iname '*.png' -o -iname '*.jpg' \) | sort
```

검사 이미지가 없으면 OCR 서비스는 서버가 실행되어도 실제 `POST /api/inspect`를 정상 처리할 수 없습니다. **교과 9에서 사용했던 이미지 6장**을 원래 경로에 배치한 뒤 진행합니다.

다음은 교과 10입니다.

```bash
ls -lh \
  ~/ai_vision/subject10_pose_action/yolo11n-pose.pt \
  ~/ai_vision/subject10_pose_action/artifacts/day06/rf_action_baseline.joblib \
  ~/ai_vision/subject10_pose_action/artifacts/day06/rf_feature_columns.json \
  ~/ai_vision/subject10_pose_action/artifacts/day06/rf_training_info.json \
  ~/ai_vision/subject10_pose_action/configs/day07_realtime.json \
  ~/ai_vision/subject10_pose_action/src/realtime_utils.py
```

한 파일이라도 `No such file or directory`로 표시된다면 해당 파일을 먼저 복구합니다. 모델 파일을 다른 이름으로 변경하거나, `rf_training_info.json`의 `sequence_steps`를 임의로 수정해서 맞추지 않습니다.

**이전 교과 10 모델의 Feature 크기를 확인합니다.** 프로젝트 최상위 폴더에서 실행합니다.

```bash
python - <<'PY'
import json
from pathlib import Path
r = Path.home() / 'ai_vision' / 'subject10_pose_action' / 'artifacts' / 'day06'
info = json.loads((r / 'rf_training_info.json').read_text(encoding='utf-8'))
columns = json.loads((r / 'rf_feature_columns.json').read_text(encoding='utf-8'))
steps = int(info['sequence_steps'])
print('Sequence Steps:', steps)
print('Frame Feature: 38')
print('Sequence Feature:', steps * 38)
print('학습 Feature Column 수:', len(columns))
assert len(columns) == steps * 38
print('Feature 확인: PASS')
PY
```

이 코드는 `joblib` 모델 자체를 실행하지 않고 **모델이 학습할 때 사용한 입력 형태가 38-D × Sequence Step인지** 확인합니다. 뒤의 Bridge에서 실제 Random Forest 파일의 `n_features_in_`도 확인합니다.

### 실습 3. 오늘 추가할 파일의 위치를 확인합니다

```bash
cd ~/ai_vision/subject13_ai_platform
mkdir -p bridges
```

기존 1~3일차 파일에 **추가하거나 변경할 파일**은 아래와 같습니다. 이전 파일을 삭제하지 않습니다.

```text
subject13_ai_platform/
├── app/
│   ├── main.py                    # 기존 파일에 Router 두 줄 추가
│   ├── schemas.py                 # source_mode에 live_inference 추가
│   ├── live_repository.py         # 새 파일: 실제 이벤트 DB 저장
│   └── live_api.py                # 새 파일: OCR·Pose 서비스 HTTP 연결
├── bridges/
│   ├── __init__.py                # 새 파일
│   └── subject10_action.py        # 새 파일: 교과10 Pose·RF Bridge
├── scripts/
│   ├── 07_check_day04_inputs.py   # 새 파일: 서비스 연결 점검
│   └── 08_verify_live_records.py  # 새 파일: PostgreSQL 저장 검증
├── tests/
│   └── test_live.py               # 새 파일: 4일차 자동 테스트
├── web/
│   └── live.html                  # 새 파일: 실시간 서비스 화면
├── .env                           # 기존 설정에 2개 URL 추가
└── .env.example                   # URL 예시 추가
```

---

## PART 2. 교과 9의 기존 OCR 서비스를 실제로 실행합니다

### 실습 4. 교과 9의 FastAPI를 8009 포트에서 실행합니다

오늘 OCR을 다시 처음부터 구현하지 않습니다. 교과 9에서 작성한 `day10_service/api_main.py`의 기존 API를 사용합니다.

```text
GET  /api/health                 서버 상태
GET  /api/samples                검사할 수 있는 이미지 목록
GET  /api/sample-image/{name}    이미지 제공
POST /api/inspect                실제 OCR 검사 후 결과 기록
```

**새 터미널 A**를 열고 교과 9 가상환경을 활성화합니다.

```bash
cd ~/ai_vision/subject9_machinevision
source .venv/bin/activate
which python
python -m pip show fastapi uvicorn pytesseract opencv-python
tesseract --version
python -m uvicorn day10_service.api_main:app --host 127.0.0.1 --port 8009
```

`pytesseract`는 Tesseract를 Python에서 호출하는 라이브러리이므로, `tesseract --version`이 실행되지 않으면 교과 9에서 설치한 시스템 OCR 엔진부터 확인합니다. `8009`는 교과 13 서버의 `8000`과 구분하기 위한 포트입니다.

**터미널 A는 종료하지 않습니다.** 다른 WSL 터미널에서 서비스를 확인합니다.

```bash
curl -s http://127.0.0.1:8009/api/health
curl -s http://127.0.0.1:8009/api/samples | python3 -m json.tool
```

이미지 목록이 비어 있으면 `samples/day06/product/`에 교과 9 원본 이미지가 있는지 확인합니다.

### 실습 5. 실제 OCR 검사를 한 번 실행합니다

교과 9에서 사용한 SN 제품 이미지 중 하나를 검사합니다. 터미널 A는 켜 둔 상태에서 실행합니다.

```bash
curl -s -X POST http://127.0.0.1:8009/api/inspect   -H 'Content-Type: application/json'   -d '{"file_name":"0637145699424_side.png"}' | python3 -m json.tool
```

이 요청은 **저장해 둔 이전 JSON을 다시 반환하는 기능이 아닙니다.** `inspect_sample()`에서 Tesseract OCR을 실행하고, `append_history()`로 교과 9의 기록에도 결과를 남깁니다.

JSON에서 `request_id`, `file_name`, `measurements.ocr_text`, `measurements.expected_text`, `final_decision`, `reasons`, `rule_version`를 확인합니다. 실제 OCR 결과는 장치 환경에 따라 달라질 수 있으므로 예측 숫자를 억지로 조작하지 않습니다. 다음 PART에서는 교과 10의 실제 Pose 모델을 연결합니다.

---

## PART 3. 교과 10의 실제 Pose·Action Bridge를 추가합니다

### 실습 6. 기존 7일차 동작인식 Pipeline을 재사용합니다

```text
스마트폰 IP Webcam / 저장 MP4
              ↓
           OpenCV Frame
              ↓
        YOLO11n-Pose 추론
              ↓
         COCO17 Keypoint
              ↓
    frame_feature_from_pose()  → 38-D
              ↓
    TimedFeatureBuffer        → 최근 N초
              ↓
    predict_from_window()     → Random Forest
              ↓
    PredictionStabilizer      → Stable Label
              ↓
       JPEG 영상 + 결과 JSON
```

Bridge는 `src/realtime_utils.py`의 함수를 **다시 구현하지 않고 import하여 사용**합니다. 그 과정에서 학습 당시 Feature의 개수(일반적으로 30 Step × 38-D = 1,140)를 실제 Training Info에서 읽으며, 모델 입력 차원이 다른 경우에는 즉시 오류를 발생시킵니다.

Bridge가 제공할 API는 `GET /bridge/health`, `POST /bridge/start`, `POST /bridge/stop`, `GET /bridge/latest`, `GET /bridge/frame.jpg`입니다. 추론은 한 개의 백그라운드 스레드에서 수행하고, FastAPI는 최근 예측과 표시용 JPEG를 제공합니다.

### 실습 7. 교과 13의 `bridges/` 안에 실제 Bridge 코드를 작성합니다

**파일: `bridges/__init__.py`**

```python
"""교과 13에서 기존 교과 AI 모델을 실행하는 연결 모듈."""
```

**의사코드**

```text
SUBJECT10_ROOT · DAY04_STREAM_SOURCE 확인
            ↓
기존 YOLO·RF 가중치·설정 파일 존재 검사
            ↓
시작 요청 시 단일 백그라운드 스레드 생성
            ↓
Pose·RF 모델을 처음 한 번 로드
            ↓
VideoCapture에서 Frame 수신
            ↓
Pose → 38-D → Sliding Window → RF → Stable Prediction
            ↓
Skeleton JPEG와 JSON을 최신 메모리에 보관
            ↓
교과13 FastAPI가 최근 JPEG·JSON 조회
            ↓
중지 버튼을 누르면 카메라 해제
```

**파일: `bridges/subject10_action.py`**

```python
"""교과10 기존 realtime_utils 재사용: 로컬 IP Webcam / MP4 Action Bridge.
이 파일은 교과 13 프로젝트의 bridges/에 두고 교과 10의 .venv로 실행합니다.
"""
import json
import os
import sys
import threading
import time
from datetime import datetime, timezone
from pathlib import Path
from uuid import uuid4

from fastapi import FastAPI, HTTPException
from fastapi.responses import Response

PROJECT_ROOT = Path(os.getenv('SUBJECT10_ROOT', str(Path.home() / 'ai_vision' / 'subject10_pose_action'))).expanduser().resolve()
# 기존 교과 10의 src.common/src.realtime_utils를 원본 위치에서 import합니다.
sys.path.insert(0, str(PROJECT_ROOT))
app = FastAPI(title='교과10 실제 Pose·Action Bridge', version='day04-1.0')
ACTION_CLASSES = {'bend_return', 'leg_raise_lower', 'walk_turn_walk',
                  'drink_return', 'sit_stand', 'wave'}


class RuntimeState:
    def __init__(self):
        self.lock = threading.Lock()
        self.stop_event = threading.Event()
        self.thread = None
        self.status = 'STOPPED'
        self.error = None
        self.latest = None
        self.jpeg = None
        self.session_id = None


state = RuntimeState()


def check_inputs():
    files = [
        PROJECT_ROOT / 'configs/project.json',
        PROJECT_ROOT / 'configs/day07_realtime.json',
        PROJECT_ROOT / 'artifacts/day06/rf_action_baseline.joblib',
        PROJECT_ROOT / 'artifacts/day06/rf_feature_columns.json',
        PROJECT_ROOT / 'artifacts/day06/rf_training_info.json',
    ]
    config_path = PROJECT_ROOT / 'configs/project.json'
    if config_path.exists():
        config = json.loads(config_path.read_text(encoding='utf-8'))
        files.append(PROJECT_ROOT / config['pose_model'])
    missing = [str(p.relative_to(PROJECT_ROOT)) for p in files if not p.is_file()]
    return missing


def publish(latest=None, jpeg=None, status=None, error=None):
    with state.lock:
        if latest is not None:
            state.latest = latest
        if jpeg is not None:
            state.jpeg = jpeg
        if status is not None:
            state.status = status
        if error is not None:
            state.error = error


def run_inference(source: str, session_id: str):
    """카메라 읽기와 모델 추론은 단 한 개의 백그라운드 스레드에서 수행합니다."""
    import cv2
    import joblib
    from ultralytics import YOLO
    from src.common import load_config, resolve_device
    from src.realtime_utils import (
        FPSMeter, PredictionStabilizer, TimedFeatureBuffer,
        frame_feature_from_pose, predict_from_window,
        validate_runtime_values,
    )

    capture = None
    try:
        project_config = load_config()
        runtime_config = json.loads((PROJECT_ROOT / 'configs/day07_realtime.json').read_text(encoding='utf-8'))
        train_info = json.loads((PROJECT_ROOT / 'artifacts/day06/rf_training_info.json').read_text(encoding='utf-8'))
        feature_names = json.loads((PROJECT_ROOT / 'artifacts/day06/rf_feature_columns.json').read_text(encoding='utf-8'))
        steps = int(train_info['sequence_steps'])
        expected_dim = steps * 38
        model = joblib.load(PROJECT_ROOT / 'artifacts/day06/rf_action_baseline.joblib')
        if (len(feature_names) != expected_dim or
                int(model.n_features_in_) != expected_dim):
            raise RuntimeError('6일차 Random Forest Feature 수 또는 순서 정보를 확인하세요.')
        cfg = runtime_config
        validate_runtime_values(
            window_seconds=float(cfg['window_seconds']),
            confidence_threshold=float(cfg['confidence_threshold']),
            stabilization_buffer=int(cfg['stabilization_buffer']),
            stabilization_min_votes=int(cfg['stabilization_min_votes']),
        )
        pose_model = YOLO(str(PROJECT_ROOT / project_config['pose_model']))
        device = resolve_device(project_config['device'])
        capture = cv2.VideoCapture(source)
        if not capture.isOpened():
            raise RuntimeError('IP Webcam/MP4를 열 수 없습니다. 주소·네트워크·파일 경로를 확인하세요.')
        buffer = TimedFeatureBuffer(float(cfg['window_seconds']))
        stabilizer = PredictionStabilizer(int(cfg['stabilization_buffer']), int(cfg['stabilization_min_votes']))
        fps_meter = FPSMeter()
        start = time.monotonic()
        last_predict = -1e9
        raw, threshold, stable, confidence = 'WARMING_UP', 'WARMING_UP', 'WARMING_UP', None
        prediction_id, prediction_time = None, None
        frame_id = 0
        publish(status='RUNNING')
        while not state.stop_event.is_set():
            ok, frame = capture.read()
            if not ok or frame is None:
                # MP4라면 재생 종료, 스트림이라면 중단 상태를 표시합니다.
                raise RuntimeError('카메라 프레임을 읽지 못했거나 영상 재생이 끝났습니다.')
            now = time.monotonic()
            elapsed = now - start
            result = pose_model.predict(
                source=frame, imgsz=int(project_config['image_size']),
                conf=float(project_config['person_confidence']),
                device=device, verbose=False,
            )[0]
            buffer.append(elapsed, frame_feature_from_pose(result))
            ready = buffer.is_ready(
                float(cfg['min_window_fill_ratio']),
                float(cfg['min_valid_feature_ratio']),
                int(cfg['min_valid_features']),
            )
            if ready and elapsed - last_predict >= float(cfg['prediction_interval_sec']):
                threshold, raw, confidence, _ = predict_from_window(
                    model=model, feature_buffer=buffer, sequence_steps=steps,
                    confidence_threshold=float(cfg['confidence_threshold']),
                )
                stable = stabilizer.update(threshold)
                prediction_id = str(uuid4())
                prediction_time = datetime.now(timezone.utc).isoformat()
                last_predict = elapsed
            elif not ready:
                raw, threshold, stable, confidence = 'WARMING_UP', 'WARMING_UP', 'WARMING_UP', None
                prediction_id, prediction_time = None, None
                stabilizer.clear()
            image = result.plot()  # YOLO Pose의 기존 BBox/Skeleton 렌더링
            fps = fps_meter.update(time.monotonic())
            panel = cv2.copyMakeBorder(image, 0, 0, 0, 370, cv2.BORDER_CONSTANT, value=(25, 25, 25))
            x0 = image.shape[1] + 12
            labels = [
                'REAL YOLO POSE + RF', f'Raw: {raw}', f'Threshold: {threshold}',
                f'Stable: {stable}', f'RF confidence: {confidence if confidence is not None else "-"}',
                f'FPS: {fps:.1f}', f'Window: {buffer.duration():.1f}s',
            ]
            for index, label in enumerate(labels):
                cv2.putText(panel, label[:43], (x0, 32 + index * 34),
                            cv2.FONT_HERSHEY_SIMPLEX, 0.52, (255, 255, 255), 1, cv2.LINE_AA)
            encoded, jpg = cv2.imencode('.jpg', panel, [int(cv2.IMWRITE_JPEG_QUALITY), 80])
            if not encoded:
                raise RuntimeError('웹 화면용 JPEG 인코딩에 실패했습니다.')
            frame_id += 1
            latest = {
                'session_id': session_id, 'frame_id': frame_id,
                'prediction_id': prediction_id, 'prediction_time_utc': prediction_time,
                'raw_label': raw, 'threshold_label': threshold, 'stable_label': stable,
                'raw_confidence': float(confidence) if confidence is not None else None,
                'sequence_steps': steps, 'valid_ratio': round(buffer.valid_ratio(), 3),
                'window_duration_sec': round(buffer.duration(), 2),
                'processing_fps': round(fps, 2),
                'source_kind': 'IP Webcam' if source.startswith(('http://', 'https://')) else 'MP4',
                'ready': ready,
            }
            publish(latest=latest, jpeg=jpg.tobytes())
    except Exception as error:
        if not state.stop_event.is_set():
            publish(status='ERROR', error=str(error))
    finally:
        if capture is not None:
            capture.release()
        with state.lock:
            if state.status != 'ERROR':
                state.status = 'STOPPED'


@app.get('/bridge/health')
def health():
    with state.lock:
        return {'status': state.status, 'error': state.error,
                'inputs_missing': check_inputs(), 'session_id': state.session_id}


@app.post('/bridge/start')
def start():
    source = os.getenv('DAY04_STREAM_SOURCE', '').strip()
    if not source:
        raise HTTPException(400, '교과10 터미널에 DAY04_STREAM_SOURCE를 설정하세요.')
    if not source.startswith(('http://', 'https://')) and not Path(source).is_file():
        raise HTTPException(400, '영상 파일의 절대경로 또는 IP Webcam /video 주소를 확인하세요.')
    missing = check_inputs()
    if missing:
        raise HTTPException(409, {'missing_files': missing})
    with state.lock:
        if state.thread is not None and state.thread.is_alive():
            raise HTTPException(409, '이미 추론 중입니다. 먼저 중지하세요.')
        state.stop_event.clear()
        state.status = 'STARTING'
        state.error, state.jpeg, state.latest = None, None, None
        state.session_id = str(uuid4())
        session_id = state.session_id
        state.thread = threading.Thread(target=run_inference, args=(source, session_id), daemon=True)
        state.thread.start()
    return {'status': 'STARTING', 'session_id': session_id}


@app.post('/bridge/stop')
def stop():
    state.stop_event.set()
    return {'status': 'STOP_REQUESTED', 'note': '프레임 읽기 중이면 중지에 시간이 걸릴 수 있습니다.'}


@app.get('/bridge/latest')
def latest():
    with state.lock:
        return {'status': state.status, 'error': state.error,
                'latest': dict(state.latest) if state.latest else None}


@app.get('/bridge/frame.jpg')
def frame():
    with state.lock:
        jpg = state.jpeg
    if jpg is None:
        raise HTTPException(503, '첫 분석 프레임을 기다리고 있습니다.')
    return Response(jpg, media_type='image/jpeg', headers={'Cache-Control': 'no-store'})
```

**코드에서 중요한 부분**

`PROJECT_ROOT`는 교과 10의 기존 프로젝트 위치입니다. 따라서 모델 가중치, Feature 코드, Config는 교과 10 원본을 그대로 사용합니다. `frame_feature_from_pose()`, `TimedFeatureBuffer`, `predict_from_window()`, `PredictionStabilizer`, `FPSMeter`도 이전 교과에서 작성한 함수를 가져옵니다.

`result.plot()`은 YOLO가 검출한 BBox·Skeleton을 표시합니다. 이 영상에 Raw Label, Stable Label, RF Confidence, FPS를 추가하여 JPEG 한 장으로 만들고 최신 프레임만 유지합니다. **실제 모델이 추론하지 않은 모의 데이터를 만들지 않습니다.**

`check_inputs()`는 필요한 모델 파일·설정이 없을 경우 오류를 표시합니다. `run_inference()`에서 오류가 발생하면 `/bridge/health`에 `ERROR`와 이유가 표시됩니다. 이 Bridge는 학습용 로컬 서비스로, 권한 인증을 구현하지 않았으므로 `127.0.0.1`로만 실행합니다.

### 실습 8. 교과 10 가상환경으로 Bridge를 실행합니다

**새 WSL 터미널 B**를 열고 다음을 실행합니다. 작업 폴더는 교과 13이지만, **Python 가상환경은 교과 10의 `.venv`**를 사용합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source ~/ai_vision/subject10_pose_action/.venv/bin/activate
which python
python -m pip show ultralytics joblib opencv-python fastapi uvicorn
```

교과 10 가상환경에 FastAPI·Uvicorn이 없다면 다음처럼 설치합니다. 다른 패키지를 한꺼번에 업그레이드하지 않습니다.

```bash
python -m pip install fastapi 'uvicorn[standard]'
```

스마트폰 IP Webcam 앱을 시작하고 스마트폰 화면에 표시된 주소를 확인합니다. 다음 `192.168.0.10`은 **예시**입니다. 실제 주소로 바꿉니다.

```bash
export SUBJECT10_ROOT="$HOME/ai_vision/subject10_pose_action"
export DAY04_STREAM_SOURCE='http://192.168.0.10:8080/video'
```

스마트폰과 PC가 통신 가능한 같은 네트워크에 있는지, Windows 브라우저에서 `http://스마트폰IP:포트`가 열리는지 확인합니다. WSL OpenCV에는 보통 `/video`가 붙은 URL을 사용합니다.

IP Webcam 연결 전에 실제 모델과 서비스를 먼저 확인하고 싶으면 충분히 긴 **기존 교과 10 MP4**로 같은 추론 코드를 실행할 수 있습니다.

```bash
export DAY04_STREAM_SOURCE="$HOME/ai_vision/subject10_pose_action/data/day05/videos/S02/S02_A003_T01.mp4"
```

이 MP4 파일이 존재해야 하며, 영상이 Sliding Window 채움 시간보다 너무 짧으면 Action 예측 전에 종료될 수 있습니다. **최종 실시간 입력 확인은 IP Webcam으로 진행합니다.**

이제 Bridge를 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
python -m uvicorn bridges.subject10_action:app --host 127.0.0.1 --port 8010
```

터미널 B는 실행한 채 둡니다. 아직 `POST /bridge/start`를 호출하지 않았으므로 카메라와 모델은 바로 시작되지 않습니다. 다른 터미널에서 준비 파일을 확인합니다.

```bash
curl -s http://127.0.0.1:8010/bridge/health | python3 -m json.tool
```

`inputs_missing: []`가 나타나야 합니다. 누락 파일이 출력되면 교과 10의 원본 프로젝트 경로를 확인하고, 필요한 산출물을 복구한 뒤 다시 진행합니다.

---

## PART 4. 공통 JSON과 PostgreSQL에 새 추론 결과를 저장합니다

### 실습 9. 기존 `InferenceEvent`의 출처 구분을 확장합니다

1일차의 `recorded_result`와 2일차의 `manual_entry`를 유지한 채, 4일차 실제 추론 결과를 나타낼 `live_inference`를 허용합니다.

**수정 파일: `app/schemas.py`**

기존 코드의 `source_mode` **한 줄만** 변경합니다.

```python
# 기존 2일차 코드
source_mode: Literal["recorded_result", "manual_entry"] = "recorded_result"

# 4일차 변경 코드: 기존 한 줄을 아래 줄로 교체
source_mode: Literal["recorded_result", "manual_entry", "live_inference"] = "recorded_result"
```

Pydantic에서 허용할 값만 추가했으며, 2일차 DB의 `source_mode` 컬럼은 이미 문자열을 저장하므로 **테이블을 삭제하거나 재생성할 필요가 없습니다.**

### 실습 10. 기존 PostgreSQL 테이블에 실제 결과 저장 기능을 추가합니다

오늘 새로 만드는 `save_live_event()`는 기존 2일차 `camera_sources`·`ai_models`·`inference_events` 테이블을 재사용합니다.

**의사코드**

```text
공통 이벤트 받기 → live_inference 확인 → event_id 중복 검사
    ↓
입력 출처 만들기/재사용 → 모델 정보 만들기/재사용
    ↓
inference_events INSERT → commit → 저장한 이벤트 반환
```

**파일: `app/live_repository.py`**

```python
"""4일차: 기존 2일차 테이블에 새 AI 추론 결과만 INSERT 합니다."""
from datetime import timezone

from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from app.models import CameraSource, InferenceEventRow
from app.repository import get_or_create_model, row_to_event
from app.schemas import InferenceEvent


def save_live_event(db: Session, event: InferenceEvent) -> tuple[InferenceEvent, bool]:
    """같은 event_id로 재요청해도 중복 저장되지 않습니다."""
    if event.source_mode != 'live_inference':
        raise ValueError('4일차에는 새 추론 결과(live_inference)만 저장합니다.')
    lookup = (select(InferenceEventRow)
              .options(selectinload(InferenceEventRow.source),
                       selectinload(InferenceEventRow.model))
              .where(InferenceEventRow.event_id == event.event_id))
    try:
        old = db.scalar(lookup)
        if old is not None:
            return row_to_event(old), False
        source_key = f'day04-{event.task}-live'
        source = db.scalar(select(CameraSource).where(CameraSource.source_key == source_key))
        if source is None:
            source = CameraSource(
                source_key=source_key,
                name='4일차 실시간 제조 OCR' if event.task == 'inspection' else '4일차 IP Webcam Pose',
                source_type='image_file' if event.task == 'inspection' else 'ip_webcam',
            )
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

이 함수는 기존 2일차의 `get_or_create_model()`을 그대로 호출합니다. 기록 시간과 DB 저장 시간을 구분하고, OCR Confidence가 없는 경우 SQL `NULL`을 유지합니다. 동일한 `event_id`를 다시 저장하면 이미 존재하는 행을 반환하고 중복 INSERT하지 않습니다.

### 실습 11. 두 교과의 실제 추론 API를 교과 13에 연결합니다

**의사코드**

```text
제조 검사 : OCR 이미지 선택 → 교과9 POST /api/inspect
                              ↓
                      inspection_to_event() (1일차 함수)
                              ↓
                      live_inference 지정 → DB 저장

동작 인식 : 교과10 GET /bridge/latest, /bridge/frame.jpg
                              ↓
                      브라우저에 Pose·Stable Action 표시
                              ↓
                기준 Action 선택 후 저장 버튼 클릭
                              ↓
                OK / NG / UNKNOWN 결정 → DB 저장
```

실제 Action 예측은 Bridge에서 이미 수행했습니다. 교과 13의 `live_api.py`는 그 결과를 읽고 저장할 뿐 교과 10 모델을 다시 학습하지 않습니다. 보안상 영상 전체를 DB에 넣지 않고, 출처 메타데이터·모델·판정·FPS 등을 기록합니다.

**파일: `app/live_api.py`**

```python
"""4일차: 교과9 OCR·교과10 Pose/Action 기존 실행환경을 HTTP로 연결합니다."""
import os
from datetime import datetime, timezone
from pathlib import Path
from typing import Literal
from uuid import UUID

import httpx
from dotenv import load_dotenv
from fastapi import APIRouter, Depends, HTTPException
from fastapi.responses import FileResponse, Response
from pydantic import BaseModel
from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from app.db import get_db
from app.live_repository import save_live_event
from app.models import InferenceEventRow
from app.records import inspection_to_event
from app.repository import row_to_event
from app.schemas import InferenceEvent

ROOT = Path(__file__).resolve().parents[1]
load_dotenv(ROOT / '.env')
OCR_URL = os.getenv('DAY04_OCR_URL', 'http://127.0.0.1:8009').rstrip('/')
ACTION_URL = os.getenv('DAY04_ACTION_URL', 'http://127.0.0.1:8010').rstrip('/')
router = APIRouter()
ACTION_LABELS = ('bend_return', 'leg_raise_lower', 'walk_turn_walk',
                 'drink_return', 'sit_stand', 'wave')


class OCRRequest(BaseModel):
    file_name: str


class SaveActionRequest(BaseModel):
    expected_action: Literal['bend_return', 'leg_raise_lower', 'walk_turn_walk',
                             'drink_return', 'sit_stand', 'wave']


def backend_request(base: str, method: str, endpoint: str, **kwargs):
    """교과 9/10 별도 프로세스 호출: 연결 실패는 503으로 안내합니다."""
    try:
        with httpx.Client(timeout=10.0) as client:
            reply = client.request(method, base + endpoint, **kwargs)
        if reply.status_code >= 400:
            # 요청 실패 이유를 보존하되 과도한 내부 로그는 외부에 내보내지 않습니다.
            raise HTTPException(status_code=502,
                                detail=f'기존 AI 서비스 {method} {endpoint} 응답 {reply.status_code}: {reply.text[:180]}')
        return reply
    except httpx.RequestError as error:
        raise HTTPException(status_code=503,
                            detail=f'기존 AI 서비스에 연결할 수 없습니다. {base} 실행 상태를 확인하세요.') from error


def bridge_latest() -> dict:
    return backend_request(ACTION_URL, 'GET', '/bridge/latest').json()


@router.get('/live')
def live_page():
    return FileResponse(ROOT / 'web' / 'live.html')


@router.get('/api/v1/live/health')
def health():
    result = {}
    for key, base, route in (
        ('inspection', OCR_URL, '/api/health'),
        ('action', ACTION_URL, '/bridge/health'),
    ):
        try:
            result[key] = {'connected': True, 'detail': backend_request(base, 'GET', route).json()}
        except HTTPException as error:
            result[key] = {'connected': False, 'detail': error.detail}
    return result


@router.get('/api/v1/live/inspection/samples')
def inspection_samples():
    return backend_request(OCR_URL, 'GET', '/api/samples').json()


@router.get('/api/v1/live/inspection/image/{file_name}')
def inspection_sample_image(file_name: str):
    samples = inspection_samples().get('samples', [])
    if file_name not in samples:
        raise HTTPException(404, '등록된 OCR 검사 이미지가 아닙니다.')
    raw = backend_request(OCR_URL, 'GET', f'/api/sample-image/{file_name}').content
    media = 'image/png' if file_name.lower().endswith('.png') else 'image/jpeg'
    return Response(raw, media_type=media, headers={'Cache-Control': 'no-store'})


@router.post('/api/v1/live/inspection/run')
def run_inspection(payload: OCRRequest, db: Session = Depends(get_db)):
    if payload.file_name not in inspection_samples().get('samples', []):
        raise HTTPException(422, '목록에 있는 이미지 이름을 선택하세요.')
    original = backend_request(OCR_URL, 'POST', '/api/inspect',
                               json={'file_name': payload.file_name}).json()
    event = inspection_to_event(original).model_copy(update={
        'source_mode': 'live_inference',
        'source_path': 'day10_service/api_main.py:POST /api/inspect',
    })
    stored, created = save_live_event(db, event)
    return {'saved': created, 'event': stored.model_dump(mode='json')}


@router.post('/api/v1/live/action/start')
def action_start():
    return backend_request(ACTION_URL, 'POST', '/bridge/start').json()


@router.post('/api/v1/live/action/stop')
def action_stop():
    return backend_request(ACTION_URL, 'POST', '/bridge/stop').json()


@router.get('/api/v1/live/action/latest')
def action_latest():
    return bridge_latest()


@router.get('/api/v1/live/action/frame.jpg')
def action_frame():
    raw = backend_request(ACTION_URL, 'GET', '/bridge/frame.jpg').content
    return Response(raw, media_type='image/jpeg', headers={'Cache-Control': 'no-store'})


@router.post('/api/v1/live/action/save')
def save_action(payload: SaveActionRequest, db: Session = Depends(get_db)):
    response = bridge_latest()
    latest = response.get('latest')
    if response.get('status') != 'RUNNING' or latest is None:
        raise HTTPException(409, 'Pose 추론을 실행하고 첫 예측 결과를 기다리세요.')
    stable = latest.get('stable_label')
    if stable not in ACTION_LABELS + ('UNKNOWN',):
        raise HTTPException(409, 'Sliding Window가 아직 준비되지 않았습니다.')
    try:
        UUID(latest['session_id'])
        UUID(latest['prediction_id'])
        stamp = datetime.fromisoformat(latest['prediction_time_utc'])
        if stamp.tzinfo is None:
            raise ValueError('시간대 정보 없음')
    except (ValueError, TypeError, KeyError) as error:
        raise HTTPException(502, '교과10 Bridge의 추론 ID/시간 형식을 확인하세요.') from error
    seconds_old = (datetime.now(timezone.utc) - stamp.astimezone(timezone.utc)).total_seconds()
    if not (-2 <= seconds_old <= 5):
        raise HTTPException(409, '마지막 예측이 오래되었습니다. 연결 상태를 확인하세요.')
    is_unknown = stable == 'UNKNOWN'
    decision = 'UNKNOWN' if is_unknown else ('OK' if stable == payload.expected_action else 'NG')
    reason = ('안정된 동작 예측을 확정하지 못했습니다.' if is_unknown else
              ('선택한 기준 동작과 일치합니다.' if decision == 'OK' else '선택한 기준 동작과 다릅니다.'))
    raw_conf = latest.get('raw_confidence')
    # RF의 최신 raw 예측 Confidence와 Stable Vote의 Confidence는 서로 다른 값입니다.
    confidence = raw_conf if stable == latest.get('raw_label') and not is_unknown else None
    event = InferenceEvent(
        event_id=f'action-live:{latest["session_id"]}:{latest["prediction_id"]}',
        task='action', source_mode='live_inference', source_project='subject10_pose_action',
        source_path='bridges/subject10_action.py:GET /bridge/latest',
        recorded_at=stamp, input_ref='ip_webcam' if latest.get('source_kind') == 'IP Webcam' else 'mp4_runtime',
        processor_id='yolo11n-pose+day06-random-forest', processor_version=None,
        predicted_label=stable, expected_label=payload.expected_action,
        confidence=confidence, decision=decision, reasons=[reason],
        details={
            'raw_label': latest['raw_label'], 'threshold_label': latest['threshold_label'],
            'stable_label': stable, 'raw_confidence': raw_conf,
            'confidence_meaning': '현재 Raw RF 클래스 확률이며 Stable Label 확률이 아닙니다.',
            'frame_id': latest['frame_id'], 'fps': latest['processing_fps'],
            'sequence_steps': latest['sequence_steps'], 'window_duration_sec': latest['window_duration_sec'],
            'session_id': latest['session_id'], 'source_kind': latest.get('source_kind'),
        },
    )
    stored, created = save_live_event(db, event)
    return {'saved': created, 'event': stored.model_dump(mode='json')}


@router.get('/api/v1/live/events', response_model=list[InferenceEvent])
def recent_live_events(db: Session = Depends(get_db)):
    stmt = (select(InferenceEventRow)
            .options(selectinload(InferenceEventRow.source), selectinload(InferenceEventRow.model))
            .where(InferenceEventRow.source_mode == 'live_inference')
            .order_by(InferenceEventRow.id.desc()).limit(30))
    return [row_to_event(item) for item in db.scalars(stmt).all()]
```

**API에서 확인할 주요 사항**

- `GET /api/v1/live/health`: 두 교과 서비스의 연결 상태
- `GET /api/v1/live/inspection/samples`: 기존 OCR 이미지 목록
- `POST /api/v1/live/inspection/run`: 실제 Tesseract OCR 검사 후 PostgreSQL 저장
- `POST /api/v1/live/action/start`, `/stop`: 실제 Pose/Action Bridge 실행/중지 요청
- `GET /api/v1/live/action/latest`, `/frame.jpg`: 최신 안정화 예측과 BBox/Skeleton 표시
- `POST /api/v1/live/action/save`: 선택한 기대 Action과 Stable Action을 비교하여 PostgreSQL 저장
- `GET /api/v1/live/events`: 새로 저장한 `live_inference` 이벤트만 조회

`WARMING_UP`은 Window 준비 단계이므로 저장을 거부하고, 결과가 5초 이상 오래되었다면 잘못된 현재 상태 저장을 방지합니다. `UNKNOWN`은 NG로 바꾸지 않습니다. `raw_confidence`는 Raw 클래스 확률이므로 Stable Label과 다른 경우 `confidence=null`을 사용합니다.

### 실습 12. 두 서버 주소를 `.env`에 추가합니다

2일차의 DB 설정과 3일차의 `REDIS_URL`을 **그대로 유지**한 채, 교과 13의 `.env.example`과 실제 `.env` 아래쪽에 다음 두 줄을 추가합니다.

```dotenv
# 4일차 OCR 및 Pose/Action 서비스 (한 PC에서 각각 실행)
DAY04_OCR_URL=http://127.0.0.1:8009
DAY04_ACTION_URL=http://127.0.0.1:8010
```

스마트폰 IP 주소는 이 `.env`에 넣지 않습니다. **터미널 B의 `DAY04_STREAM_SOURCE`**에서 지정합니다. `.env`에는 DB 비밀번호가 들어 있으므로 Git에 추가하지 않습니다.

### 실습 13. 기존 `app/main.py`에 4일차 Router를 연결합니다

**1~3일차 Router를 삭제하지 않습니다.** `app/main.py` 맨 아래에 다음 내용이 순서대로 있어야 합니다. 중복해서 추가하지 않습니다.

```python
# 2일차
from app.db_api import router as db_router
app.include_router(db_router)

# 3일차
from app.pipeline_api import router as pipeline_router
app.include_router(pipeline_router)

# 4일차 — 새로 추가할 두 줄
from app.live_api import router as live_router
app.include_router(live_router)
```

FastAPI 앱을 새로 생성하지 않습니다. 1일차부터 만든 동일한 `app = FastAPI(...)`에 연결합니다.

### 실습 14. 서비스 연결 검사 스크립트를 작성합니다

**의사코드:** PostgreSQL `SELECT 1` 확인 → 교과 9 `/api/health` 조회 → 교과 10 `/bridge/health` 조회 → 세 구성 요소가 준비되면 READY 출력.

**파일: `scripts/07_check_day04_inputs.py`**

```python
"""4일차: 1~3일차 DB와 교과9·10 독립 서버의 실행 상태를 확인합니다."""
import json
import os
from pathlib import Path

import httpx
from dotenv import load_dotenv
from sqlalchemy import func, select, text

from app.db import SessionLocal
from app.models import InferenceEventRow

ROOT = Path(__file__).resolve().parents[1]
load_dotenv(ROOT / '.env')


def check_server(label, url, path):
    try:
        response = httpx.get(url.rstrip('/') + path, timeout=4.0)
        response.raise_for_status()
        data = response.json()
        print(f'{label}: HTTP {response.status_code} / {json.dumps(data, ensure_ascii=False)[:180]}')
        return True
    except (httpx.HTTPError, ValueError) as error:
        print(f'[확인 필요] {label}: {type(error).__name__}')
        return False


def main():
    with SessionLocal() as db:
        db.execute(text('SELECT 1'))
        count = db.scalar(select(func.count()).select_from(InferenceEventRow))
        print(f'PostgreSQL 연결: OK / 저장 이벤트 {count}건')
    ocr = check_server('교과9 OCR', os.getenv('DAY04_OCR_URL', 'http://127.0.0.1:8009'), '/api/health')
    action = check_server('교과10 Pose/Action', os.getenv('DAY04_ACTION_URL', 'http://127.0.0.1:8010'), '/bridge/health')
    if not ocr or not action:
        raise SystemExit('교과9·10 서비스 터미널과 포트를 확인하고 다시 실행하세요.')
    print('4일차 기본 서비스 연결: READY')


if __name__ == '__main__':
    main()
```

**실행합니다.** 터미널 A의 OCR 서버와 터미널 B의 Bridge가 실행 중이어야 합니다. 다른 터미널에서 교과 13 가상환경을 활성화합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m scripts.07_check_day04_inputs
```

`PostgreSQL 연결: OK`, `교과9 OCR: HTTP 200`, `교과10 Pose/Action: HTTP 200`, `4일차 기본 서비스 연결: READY`가 나타나면 다음 단계로 진행합니다. 이 상태는 **서버 연결 준비**를 확인한 것이며, 아직 카메라로 실제 Pose 추론을 했다는 뜻은 아닙니다.

---

## PART 5. 통합 웹 화면에 OCR·Pose/Action 결과를 표시합니다

### 실습 15. 실시간 결과 조회 화면을 작성합니다

이전 웹 페이지를 삭제하지 않고 새 파일 `web/live.html`을 작성합니다. 다음처럼 각 영역의 목적을 분리합니다.

```text
[교과9 OCR 영역]                     [교과10 Pose·Action 영역]
검사 이미지 목록 요청                 Bridge 추론 시작 버튼
        ↓                                     ↓
실제 제품 이미지 표시               최신 BBox·Skeleton JPEG 표시
        ↓                                     ↓
실제 OCR 검사 및 DB 저장               Raw·Threshold·Stable·FPS 표시
        ↓                                     ↓
공통 JSON 확인                        기준 Action 선택 및 DB 저장
        └─────────────┬───────────────────────┘
                      ↓
               새 저장 이벤트 목록
                      ↓
                 상세 JSON 조회
```

**웹 코드의 처리 순서(의사코드)**

```text
브라우저가 /live 접속
       ↓
교과13 FastAPI에서 HTML 제공
       ↓
fetch()가 교과13 /api/v1/live/... 호출
       ↓
교과13가 교과9·10의 각 서버에서 실제 결과를 받음
       ↓
브라우저에 JSON·JPEG 표시
       ↓
저장 버튼을 누르면 PostgreSQL에 새로운 이벤트 기록
```

**파일: `web/live.html`**

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>교과 13 · 4일차 실제 AI 서비스</title>
  <style>
    body{font-family:system-ui,sans-serif;max-width:1120px;margin:26px auto;padding:0 16px;color:#182d42}
    section{border:1px solid #ccd6e0;border-radius:10px;padding:18px;margin:14px 0}
    button,select{font:inherit;padding:9px;margin:5px 5px 5px 0}
    button{cursor:pointer} .split{display:grid;grid-template-columns:1fr 1fr;gap:14px}
    img{max-width:100%;border:1px solid #dde3e8;min-height:100px}
    pre{white-space:pre-wrap;overflow-wrap:anywhere;background:#f5f7f9;padding:12px;border-radius:5px}
    table{width:100%;border-collapse:collapse}td,th{border-bottom:1px solid #e0e5ea;padding:8px;text-align:left}
    .note{color:#53677a;font-size:.92rem}.error{color:#b4232e} .ok{color:green}
    @media(max-width:800px){.split{grid-template-columns:1fr}}
  </style>
</head>
<body>
  <h1>교과 13 · 4일차 실제 AI 플랫폼</h1>
  <p>1~3일차 공통 JSON·PostgreSQL·FastAPI에 <b>교과 9 OCR</b>과 <b>교과 10 실시간 Pose·Action</b>을 연결합니다.</p>
  <section>
    <h2>0. 서비스 연결 확인</h2>
    <button id="healthBtn">교과 9·10 연결 확인</button>
    <pre id="health">서버 실행 상태를 확인하세요.</pre>
    <p class="note">OCR과 Pose 서버는 각각 교과 9·10의 가상환경에서 실행합니다. 이 화면은 교과 13의 서버입니다.</p>
  </section>
  <div class="split">
    <section>
      <h2>1. 교과 9 · 실제 OCR 검사</h2>
      <button id="samplesBtn">검사 이미지 목록 가져오기</button>
      <select id="samples"><option value="">이미지 선택</option></select>
      <button id="inspectBtn">실제 OCR 검사 및 DB 저장</button>
      <p id="inspectMessage" class="note"></p>
      <img id="ocrImage" alt="선택한 OCR 제품 이미지">
      <pre id="ocrResult">검사 버튼을 눌러주세요.</pre>
    </section>
    <section>
      <h2>2. 교과 10 · IP Webcam Pose + Action</h2>
      <button id="startBtn">실제 추론 시작</button>
      <button id="stopBtn">추론 중지</button>
      <p class="note">첫 번째 예측에는 Sliding Window가 채워질 시간이 필요합니다. 아래 영상은 YOLO Pose가 표시한 BBox·Skeleton입니다.</p>
      <img id="poseImage" alt="실시간 YOLO Pose Skeleton 프레임">
      <pre id="poseStatus">대기 중</pre>
      <label for="target">기준 Action:</label>
      <select id="target">
        <option value="bend_return">bend_return</option>
        <option value="leg_raise_lower">leg_raise_lower</option>
        <option value="walk_turn_walk">walk_turn_walk</option>
        <option value="drink_return">drink_return</option>
        <option value="sit_stand">sit_stand</option>
        <option value="wave" selected>wave</option>
      </select>
      <button id="saveBtn">현재 안정화된 예측 DB 저장</button>
      <p id="actionMessage" class="note"></p>
      <pre id="actionResult">아직 저장하지 않았습니다.</pre>
    </section>
  </div>
  <section>
    <h2>3. PostgreSQL에 새로 저장한 결과</h2>
    <button id="refreshBtn">새 결과 조회</button>
    <table><thead><tr><th>구분</th><th>입력</th><th>결과</th><th>판정</th><th>상세</th></tr></thead><tbody id="events"></tbody></table>
    <pre id="details">선택한 이벤트의 JSON을 확인합니다.</pre>
  </section>
  <script>
    const $ = id => document.getElementById(id);
    async function jsonRequest(url, method='GET', data=undefined){
      const response=await fetch(url,{method,headers:data?{'Content-Type':'application/json'}:{},body:data?JSON.stringify(data):undefined});
      const parsed=await response.json();
      if(!response.ok)throw new Error(JSON.stringify(parsed));
      return parsed;
    }
    function message(id,text,error=false){$(id).textContent=text;$(id).className=error?'error':'note';}
    async function health(){
      try{$('health').textContent=JSON.stringify(await jsonRequest('/api/v1/live/health'),null,2)}
      catch(e){message('health',e.message,true)}
    }
    async function samples(){
      try{
        const data=await jsonRequest('/api/v1/live/inspection/samples');
        $('samples').replaceChildren(new Option('이미지 선택',''));
        for(const name of data.samples){$('samples').add(new Option(name,name));}
        message('inspectMessage',`검사 이미지 ${data.samples.length}장 확인`);
      }catch(e){message('inspectMessage',e.message,true)}
    }
    $('samples').addEventListener('change',()=>{
      const name=$('samples').value;
      $('ocrImage').removeAttribute('src');
      if(name){$('ocrImage').src='/api/v1/live/inspection/image/'+encodeURIComponent(name);}
    });
    async function inspect(){
      const name=$('samples').value;
      if(!name){message('inspectMessage','먼저 이미지를 선택하세요.',true);return}
      try{
        const result=await jsonRequest('/api/v1/live/inspection/run','POST',{file_name:name});
        $('ocrResult').textContent=JSON.stringify(result.event,null,2);
        message('inspectMessage',result.saved?'실제 OCR 결과를 DB에 저장했습니다.':'이미 저장한 결과입니다.');
        await events();
      }catch(e){message('inspectMessage',e.message,true)}
    }
    let latestPrediction=null;
    async function pollPose(){
      try{
        const data=await jsonRequest('/api/v1/live/action/latest');
        latestPrediction=data.latest;
        $('poseStatus').textContent=JSON.stringify(data,null,2);
        if(data.latest){$('poseImage').src='/api/v1/live/action/frame.jpg?t='+Date.now()}
      }catch(e){$('poseStatus').textContent='Pose 서버 확인: '+e.message}
    }
    async function bridgeControl(verb){
      try{
        const data=await jsonRequest('/api/v1/live/action/'+verb,'POST');
        message('actionMessage',JSON.stringify(data));
        await pollPose();
      }catch(e){message('actionMessage',e.message,true)}
    }
    async function saveAction(){
      try{
        if(!latestPrediction || !latestPrediction.ready){throw new Error('Sliding Window가 준비된 뒤 저장하세요.')}
        const result=await jsonRequest('/api/v1/live/action/save','POST',{expected_action:$('target').value});
        $('actionResult').textContent=JSON.stringify(result.event,null,2);
        message('actionMessage',result.saved?'새 실제 Action 결과를 DB에 저장했습니다.':'이미 저장한 예측 ID입니다.');
        await events();
      }catch(e){message('actionMessage',e.message,true)}
    }
    async function events(){
      try{
        const data=await jsonRequest('/api/v1/live/events');
        $('events').replaceChildren();
        for(const event of data){
          const row=document.createElement('tr');
          for(const value of [event.task,event.input_ref,event.predicted_label||'UNKNOWN',event.decision]){
            const cell=document.createElement('td');cell.textContent=value;row.appendChild(cell);
          }
          const cell=document.createElement('td');const button=document.createElement('button');
          button.textContent='JSON 보기';button.onclick=()=>{$('details').textContent=JSON.stringify(event,null,2)};
          cell.appendChild(button);row.appendChild(cell);$('events').appendChild(row);
        }
      }catch(e){$('details').textContent=e.message}
    }
    $('healthBtn').onclick=health;$('samplesBtn').onclick=samples;$('inspectBtn').onclick=inspect;
    $('startBtn').onclick=()=>bridgeControl('start');$('stopBtn').onclick=()=>bridgeControl('stop');
    $('saveBtn').onclick=saveAction;$('refreshBtn').onclick=events;
    health();events();setInterval(pollPose,1500);
  </script>
</body>
</html>
```

**HTML·JavaScript 코드 이해하기**

`fetch()`는 교과 13의 `/api/v1/live/...`를 호출하며, 교과 9·10의 별도 포트를 브라우저에서 직접 호출하지 않습니다. 같은 출처(`8000`)의 API만 사용하기 때문에 이 실습에서 별도의 CORS 설정이 필요하지 않습니다.

`pollPose()`는 약 1.5초마다 새로운 Action JSON을 요청합니다. `poseImage`는 `/frame.jpg`의 최신 JPEG를 표시하므로 BBox·Skeleton 영상과 Action 결과가 같은 화면에서 보입니다. `Date.now()`는 브라우저의 이미지 캐시 문제를 피하기 위한 값입니다.

**카메라 Frame을 받았다고 바로 Action 예측이 가능한 것은 아닙니다.** 교과 10의 Window가 충분히 채워져야 `ready=true`가 되고, 이후 Stable Label을 확인할 수 있습니다.

### 실습 16. 세 번째 터미널에서 교과 13 FastAPI를 실행합니다

**새 터미널 C**를 열고 교과 13 가상환경을 활성화합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
which python
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
```

현재 서버는 모두 세 개입니다.

| 터미널 | 활성화한 가상환경 | 포트 | 맡은 작업 |
|---|---|---:|---|
| A | 교과 9 `.venv` | `8009` | 실제 SN OCR 검사 |
| B | 교과 10 `.venv` | `8010` | YOLO Pose·Random Forest Action |
| C | 교과 13 `.venv` | `8000` | 통합 웹·공통 JSON·PostgreSQL |

Windows 브라우저에서 다음 주소를 엽니다.

```text
http://127.0.0.1:8000/live
```

이 교재는 세 서버가 **한 대의 WSL 개발 PC에서 실행되는 로컬 실습**을 기준으로 합니다. 다른 PC에서 서로 접속하는 구조는 네트워크와 접근 권한을 따로 구성해야 합니다.

### 실습 17. 교과 9의 실제 OCR 결과를 DB에 저장합니다

웹 화면에서 다음 순서대로 수행합니다.

1. **교과 9·10 연결 확인**으로 두 서버의 응답을 확인합니다.
2. **검사 이미지 목록 가져오기**를 눌러 교과 9의 제품 파일 목록을 가져옵니다.
3. 이미지 하나를 선택합니다. 예: `0637145699424_side.png`.
4. **실제 OCR 검사 및 DB 저장** 버튼을 누릅니다.
5. JSON에서 `input_ref`, `predicted_label`, `expected_label`, `decision`, `reasons`, `source_mode`를 확인합니다.

이번 결과의 구조는 다음과 같습니다. **아래 예시는 필드 형태를 설명하는 것이며 실제 OCR 값은 각자의 실행 결과**를 따라야 합니다.

```json
{
  "event_id": "inspection:실제_request_id",
  "task": "inspection",
  "source_mode": "live_inference",
  "source_project": "subject9_openCV",
  "predicted_label": "실제_OCR_SN",
  "expected_label": "Expected_SN",
  "confidence": null,
  "decision": "OK 또는 NG",
  "reasons": []
}
```

실제 코드는 기존 교과 9의 `POST /api/inspect`를 호출합니다. 고정된 예시 OK/NG 결과를 만들어 보낸 것이 아닙니다. 원본 `request_id`마다 새 기록을 저장하므로 같은 이미지를 여러 번 새로 검사하면 새 이벤트가 생성될 수 있습니다.

### 실습 18. IP Webcam의 실제 Pose·Action을 웹에 표시합니다

같은 `/live` 화면의 오른쪽 **실제 추론 시작** 버튼을 누릅니다. 최초에는 `STARTING` 상태로 표시될 수 있습니다. 모델을 로딩한 후 영상이 정상으로 들어오면 YOLO의 BBox·Skeleton 이미지가 나타납니다.

확인할 항목은 다음과 같습니다.

```text
Raw Label        : 가장 최근 RF의 직접 예측 클래스
Threshold Label  : 신뢰도 기준을 적용한 결과
Stable Label     : 안정화 Buffer 투표 결과
Raw Confidence   : 최신 Raw 예측의 클래스 확률
FPS              : 실시간 Pose·Action 처리 속도
Window Duration  : Sliding Window에 모인 Frame 시간
Ready            : 실제 분류를 시작할 만큼 입력이 준비됐는가?
```

`ready=false`, `WARMING_UP`은 Window가 아직 채워지지 않은 상태입니다. 사람이 화면에 잘 보이는지 확인하고 몇 초 기다립니다. `UNKNOWN`은 입력 품질이나 신뢰도 때문에 **Action을 확정하지 못한 상태**입니다. 이 둘을 NG로 해석하지 않습니다.

기준 Action을 선택하고 해당 동작을 수행해 봅니다. 예를 들어 `wave`를 선택한 뒤 손을 흔듭니다. 안정화된 예측이 `wave`일 때 저장하면 `OK`, 다른 동작이면 `NG`, 확정하지 못한 경우 `UNKNOWN`으로 저장합니다. 이는 **기대 동작과 비교한 교육용 결과**이며 산업안전 사고 판정이 아닙니다.

Stable Label이 나타나면 **현재 안정화된 예측 DB 저장** 버튼을 누릅니다. 반환된 JSON의 다음 필드를 확인합니다.

```text
task              : action
source_mode       : live_inference
source_project    : subject10_pose_action
expected_label    : 화면에서 선택한 기준 동작
predicted_label   : Stable Action
confidence        : 최신 Raw 클래스와 Stable Label이 일치할 때만 기록
reasons           : 일치 / 불일치 / 판정 유보의 이유
details.raw_label : 최근 Raw Action
details.fps       : 추론 처리 FPS
```

동일한 `prediction_id`를 여러 번 저장하려고 하면 DB에 같은 행이 추가되지 않습니다. 새로운 Prediction이 실제로 수행되면 다른 ID가 생성될 수 있습니다.

영상이나 예측이 멈춘 경우에는 **터미널 B의 Bridge 로그**와 `http://127.0.0.1:8010/bridge/health`부터 확인하세요. 수업이 끝나면 **추론 중지** 버튼을 눌러 카메라를 해제합니다. 네트워크에 따라 `VideoCapture.read()`가 잠시 대기 중이라 중지가 늦을 수 있습니다.

### 실습 19. 새 이벤트와 이전 DB 기록을 구분합니다

화면 아래쪽의 **새 결과 조회** 버튼을 누릅니다. `/api/v1/live/events`는 오늘 저장한 `live_inference` 이벤트만 반환합니다.

다른 DB 화면도 열어봅니다.

```text
http://127.0.0.1:8000/db
```

기존 2일차의 `/api/v1/db/events`는 PostgreSQL 테이블 전체를 읽기 때문에 `recorded_result`, `manual_entry`, `live_inference`가 함께 표시됩니다. 반면 1일차의 `/api/v1/events`는 과거에 인수한 원본 JSON을 조회합니다. **어떤 결과가 실제 새 AI 추론인지 `source_mode`로 구분**합니다.

---

## PART 6. PostgreSQL 결과와 자동 테스트를 검증합니다

### 실습 20. 새로 저장한 AI 이벤트 수를 확인합니다

**의사코드**

```text
PostgreSQL Session 열기
      ↓
source_mode = live_inference 조회
      ↓
inspection/action + OK/NG/UNKNOWN별 건수 집계
      ↓
화면에 출력
```

**파일: `scripts/08_verify_live_records.py`**

```python
"""4일차: 새 실시간 추론 결과가 PostgreSQL에 저장되었는지 확인합니다."""
from sqlalchemy import func, select

from app.db import SessionLocal
from app.models import InferenceEventRow


def main():
    with SessionLocal() as db:
        rows = db.execute(
            select(InferenceEventRow.task, InferenceEventRow.decision, func.count())
            .where(InferenceEventRow.source_mode == 'live_inference')
            .group_by(InferenceEventRow.task, InferenceEventRow.decision)
            .order_by(InferenceEventRow.task, InferenceEventRow.decision)
        ).all()
    if not rows:
        print('live_inference 기록 0건: OCR 검사 또는 Action 예측 저장을 수행하세요.')
    for task, decision, count in rows:
        print(f'{task:12s} / {decision:7s} / {count}건')
    print('주의: 위 기록은 저장된 예측·검사 결과이며 정확도 평가 지표가 아닙니다.')


if __name__ == '__main__':
    main()
```

4일차 OCR과 Action을 각각 한 번 저장한 뒤 교과 13 가상환경에서 다음 명령을 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m scripts.08_verify_live_records
```

새 기록을 저장했다면 `inspection`, `action`의 결과가 표시되어야 합니다. 아직 저장하지 않았다면 0건이라고 안내합니다. **이 숫자는 분석 기록 건수이지 AI 모델 정확도 지표가 아닙니다.**

PostgreSQL에서 직접 확인하려면 다음처럼 접속합니다.

```bash
psql -h 127.0.0.1 -U subject13_user -d subject13_platform
```

```sql
SELECT source_mode, task, decision, COUNT(*)
FROM inference_events
GROUP BY source_mode, task, decision
ORDER BY source_mode, task, decision;
```

교과 9·10 과거 결과와 오늘 새로 생성한 결과가 구분되는지 확인합니다.

### 실습 21. 자동 테스트로 실제 결과 저장 규칙을 확인합니다

이번 테스트는 **교과 9·10의 실제 응답 형식에 맞춘 시험용 HTTP 응답**으로, 교과 13의 연결과 DB 규칙을 검사합니다. 모델 파일과 카메라가 없는 환경에서도 재현 가능하도록 구성했습니다. 따라서 실제 카메라 성능은 앞의 실습 17·18에서 반드시 별도로 확인해야 합니다.

검사할 핵심 내용은 다음과 같습니다.

- OCR 결과를 공통 JSON으로 바꿀 때 원본 `decision`, `reasons`, `confidence=null`이 유지되는가?
- 동일 `event_id`는 중복 저장되지 않는가?
- Stable Action의 OK/NG와 UNKNOWN을 서로 구분하는가?
- Stable Label이 Raw Label과 다른 경우 Confidence를 꾸며내지 않는가?
- `WARMING_UP`이나 5초 이상 오래된 Prediction의 저장을 거부하는가?
- 실제 AI 결과는 2일차 수동 CRUD API로 삭제할 수 없도록 보호되는가?
- 기존 `/`, `/db`, `/pipeline` 화면은 수정하지 않고 유지되는가?

**의사코드**

```text
테스트 전용 SQLite 테이블 준비
      ↓
교과9·10의 예상 API 응답을 테스트 함수에서 생성
      ↓
FastAPI TestClient로 OCR·Action 저장 요청
      ↓
DB 이벤트 확인 / 중복·UNKNOWN·오류 검증
      ↓
테스트 끝나면 임시 DB 삭제
```

**파일: `tests/test_live.py`**

```python
"""4일차: 강의실 AI 모델 없이 테스트용 서버 응답으로 API·DB 연결 검증."""
from datetime import datetime, timedelta, timezone
from uuid import uuid4

import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.db import Base, get_db
from app.main import app
from app import live_api
from app.models import InferenceEventRow
from sqlalchemy import select
from app.records import _read_json, INSPECTION_FILE


@pytest.fixture
def live_context(tmp_path, monkeypatch):
    engine = create_engine(f'sqlite+pysqlite:///{tmp_path / "day04.sqlite"}')
    Base.metadata.create_all(engine)
    factory = sessionmaker(bind=engine, expire_on_commit=False)
    def override():
        with factory() as session:
            yield session
    app.dependency_overrides[get_db] = override
    state = {'status': 'RUNNING', 'stable': 'wave', 'raw': 'wave', 'age': 0,
             'prediction_id': str(uuid4()), 'session_id': str(uuid4())}
    sample = _read_json(INSPECTION_FILE)[0]

    class FakeResponse:
        def __init__(self, data, content=b'jpeg'):
            self._data = data
            self.content = content
        def json(self):
            return self._data

    def fake_backend(base, method, endpoint, **kwargs):
        if endpoint == '/api/health':
            return FakeResponse({'status': 'ok'})
        if endpoint == '/bridge/health':
            return FakeResponse({'status': state['status'], 'inputs_missing': []})
        if endpoint == '/api/samples':
            return FakeResponse({'samples': [sample['file_name']]})
        if endpoint.startswith('/api/sample-image/'):
            return FakeResponse({}, b'fake-img')
        if endpoint == '/api/inspect':
            return FakeResponse(sample)
        if endpoint == '/bridge/start':
            return FakeResponse({'status': 'STARTING'})
        if endpoint == '/bridge/stop':
            return FakeResponse({'status': 'STOP_REQUESTED'})
        if endpoint == '/bridge/frame.jpg':
            return FakeResponse({}, b'jpeg')
        if endpoint == '/bridge/latest':
            stamp = datetime.now(timezone.utc) - timedelta(seconds=state['age'])
            return FakeResponse({'status': state['status'], 'latest': {
                'session_id': state['session_id'],
                'prediction_id': state['prediction_id'],
                'prediction_time_utc': stamp.isoformat(),
                'ready': True, 'stable_label': state['stable'], 'raw_label': state['raw'],
                'threshold_label': state['raw'], 'raw_confidence': .89,
                'frame_id': 82, 'processing_fps': 8.4,
                'sequence_steps': 30, 'window_duration_sec': 4.1,
                'source_kind': 'IP Webcam',
            }})
        raise AssertionError(endpoint)
    monkeypatch.setattr(live_api, 'backend_request', fake_backend)
    try:
        with TestClient(app) as client:
            yield client, factory, state, sample
    finally:
        app.dependency_overrides.clear()
        engine.dispose()


def test_health_and_live_page(live_context):
    client, _, _, _ = live_context
    assert client.get('/live').status_code == 200
    checks = client.get('/api/v1/live/health').json()
    assert checks['inspection']['connected'] is True
    assert checks['action']['connected'] is True


def test_sample_image_and_invalid_name(live_context):
    client, _, _, sample = live_context
    name = sample['file_name']
    assert client.get('/api/v1/live/inspection/samples').json()['samples'] == [name]
    assert client.get(f'/api/v1/live/inspection/image/{name}').status_code == 200
    assert client.get('/api/v1/live/inspection/image/unknown.png').status_code == 404


def test_actual_ocr_response_saved_with_null_confidence(live_context):
    client, factory, _, sample = live_context
    result = client.post('/api/v1/live/inspection/run', json={'file_name': sample['file_name']})
    assert result.status_code == 200, result.text
    data = result.json()
    assert data['saved'] is True
    assert data['event']['source_mode'] == 'live_inference'
    assert data['event']['confidence'] is None
    assert data['event']['decision'] == sample['final_decision']
    with factory() as db:
        rows = db.scalars(select(InferenceEventRow)).all()
        assert len(rows) == 1 and rows[0].source_mode == 'live_inference'


def test_ocr_duplicate_not_inserted_twice(live_context):
    client, _, _, sample = live_context
    payload = {'file_name': sample['file_name']}
    assert client.post('/api/v1/live/inspection/run', json=payload).json()['saved'] is True
    assert client.post('/api/v1/live/inspection/run', json=payload).json()['saved'] is False
    assert len(client.get('/api/v1/live/events').json()) == 1


def test_action_prediction_saved_with_expected_action(live_context):
    client, _, state, _ = live_context
    result = client.post('/api/v1/live/action/save', json={'expected_action': 'wave'})
    assert result.status_code == 200, result.text
    event = result.json()['event']
    assert event['decision'] == 'OK'
    assert event['predicted_label'] == 'wave'
    assert event['confidence'] == .89
    assert event['source_mode'] == 'live_inference'
    assert client.post('/api/v1/live/action/save', json={'expected_action': 'wave'}).json()['saved'] is False


def test_unknown_and_unmatched_action(live_context):
    client, _, state, _ = live_context
    state['stable'] = 'UNKNOWN'
    response = client.post('/api/v1/live/action/save', json={'expected_action': 'wave'})
    assert response.json()['event']['decision'] == 'UNKNOWN'
    assert response.json()['event']['confidence'] is None
    state['prediction_id'] = str(uuid4())
    state['stable'] = 'wave'
    result = client.post('/api/v1/live/action/save', json={'expected_action': 'sit_stand'})
    assert result.json()['event']['decision'] == 'NG'


def test_stabilizer_confidence_not_invented(live_context):
    client, _, state, _ = live_context
    state['raw'] = 'sit_stand'
    response = client.post('/api/v1/live/action/save', json={'expected_action': 'wave'})
    assert response.json()['event']['confidence'] is None
    assert response.json()['event']['details']['raw_confidence'] == .89


def test_stale_frame_and_warming_up_rejected(live_context):
    client, _, state, _ = live_context
    state['age'] = 20
    assert client.post('/api/v1/live/action/save', json={'expected_action': 'wave'}).status_code == 409
    state['age'] = 0
    state['stable'] = 'WARMING_UP'
    assert client.post('/api/v1/live/action/save', json={'expected_action': 'wave'}).status_code == 409


def test_action_invalid_expected_rejected(live_context):
    client, _, _, _ = live_context
    assert client.post('/api/v1/live/action/save', json={'expected_action': 'unsafe'}).status_code == 422


def test_action_proxy_controls(live_context):
    client, _, _, _ = live_context
    assert client.post('/api/v1/live/action/start').status_code == 200
    assert client.post('/api/v1/live/action/stop').status_code == 200
    assert client.get('/api/v1/live/action/frame.jpg').status_code == 200


def test_nonmanual_live_record_cannot_be_deleted(live_context):
    client, _, _, sample = live_context
    saved = client.post('/api/v1/live/inspection/run', json={'file_name': sample['file_name']}).json()['event']
    assert client.delete('/api/v1/db/events/' + saved['event_id']).status_code == 403
```

**실행 명령**

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m pytest -q
```

4일차 테스트는 **11개**를 추가합니다. 1일차 12개, 2일차 7개, 3일차 7개를 유지하고 있다면 전체 **37개**의 테스트를 실행하게 됩니다. 학생별로 기존 교재 수정 내용에 따라 총개수는 달라질 수 있습니다.

**자동 테스트 통과 = 실제 GPU/카메라 검증 통과**는 아닙니다. 실제 모델·카메라·Tesseract는 학생 PC에서 별도 실행해 확인합니다.

### 실습 22. 오류를 발생 단계별로 확인합니다

| 증상 | 확인할 원인 | 해결 방법 |
|---|---|---|
| OCR 서버(8009) 접속 실패 | 터미널 A 종료·포트 문제 | 교과 9 `.venv`에서 서버 실행 확인 |
| OCR 검사 이미지 없음 | `samples/day06/product/` 비어 있음 | 교과 9 원본 이미지 6장 배치 |
| `TesseractNotFoundError` | OS Tesseract 엔진 미설치 | `tesseract --version` 확인 |
| OCR NG 또는 OCR 실패 | ROI·이미지·Frozen Rule 조건 | 교과 9의 원본 설정과 입력 확인, 결과를 조작하지 않음 |
| Bridge(8010) 접속 실패 | 터미널 B 종료 | 교과 10 `.venv`로 Bridge 다시 실행 |
| `inputs_missing` | 모델·설정·가중치 누락 | 교과 10의 학습 산출물 복구 |
| 모델 로딩 후 `ERROR` | CUDA/Ultralytics/모델 버전 | Bridge 터미널 B와 `/bridge/health` 확인 |
| IP Webcam Stream 실패 | 주소·포트·네트워크·`/video` | 스마트폰 앱 → Windows 브라우저 → WSL 순서로 확인 |
| 계속 `WARMING_UP` | 유효 Pose 부족·Window 미충족 | 사람 전체가 잘 보이도록 하고 충분히 기다리기 |
| Action 저장 HTTP 409 | Window 준비 안 됨·오래된 Prediction | 최신 안정화 결과를 확인한 뒤 저장 |
| RF Feature 차원 불일치 | 모델과 Feature 목록/Training Info 불일치 | 동일한 6일차 학습 결과 세 파일 사용 |
| DB 연결 실패 | PostgreSQL 또는 `.env` 정보 | 2일차 `/api/v1/db/health`부터 확인 |
| `/live/events` 0건 | 아직 저장하지 않음·저장 API 실패 | 저장 버튼과 터미널 C 오류 확인 |
| `No module named app/bridges` | 프로젝트 루트가 아닌 위치에서 실행 | `cd ~/ai_vision/subject13_ai_platform` 후 실행 |
| 포트 중복 사용 | 8000·8009·8010 다른 프로세스 점유 | 각 서버의 기존 인스턴스 확인 후 중복 종료 |

문제가 생기면 **기존 데이터·모델 준비 → 교과 9 OCR 단독 테스트 → 교과 10 Pose 단독 테스트 → PostgreSQL → 교과 13 API → 브라우저** 순서대로 확인합니다. 오류가 난 한 단계를 먼저 찾아야 불필요한 코드를 다시 만들지 않습니다.

---

## PART 7. README·구조도·Git을 정리합니다

### 실습 23. 기존 구조도에 오늘 연결한 기능을 추가합니다

앞에서 작성한 `docs/architecture.md`의 끝에 다음 내용을 **추가**합니다. 1~3일차 내용을 지우지 않습니다.

**파일: `docs/architecture.md` — 맨 아래 추가**

````markdown
## 4일차 — 실제 머신비전 OCR·Pose/Action 연결

교과9 실제 OCR :8009 ────────────────┐
                                    ├→ 교과13 FastAPI :8000 → 공통 이벤트 → PostgreSQL
교과10 Pose·RF Bridge :8010 ────────┘                            ↓
                                                            /live 웹 화면

- 제조 검사 입력: 교과 9의 기존 SN 제품 이미지
- 동작인식 입력: IP Webcam (연결 시험용으로 저장 MP4 가능)
- 실제 추론 이벤트: `source_mode=live_inference`
- 교과 9 원본 OCR Rule을 수정하지 않음
- 교과 10의 38-D Feature·Sliding Window·RF 모델을 그대로 사용
- 공통 DB 테이블 `camera_sources`, `ai_models`, `inference_events` 재사용
- `/live`는 현재 실제 OCR·Action을 실행/확인하는 화면
- 기존 `/`, `/db`, `/pipeline` 기능 그대로 유지
- Action NG는 기준 동작 불일치이며 위험행동 확정이 아님

5일차: TinyCNN·ONNX 기반 Edge 모델 준비
````

같은 방식으로 프로젝트의 `README.md` 끝에 **세 터미널 실행 명령**을 기록합니다.

**파일: `README.md` — 맨 아래 추가**

````markdown
## 4일차: 실제 AI 연결

### 터미널 A — 교과 9 OCR (8009)
```bash
cd ~/ai_vision/subject9_machinevision
source .venv/bin/activate
python -m uvicorn day10_service.api_main:app --host 127.0.0.1 --port 8009
```

### 터미널 B — 교과 10 Pose/Action Bridge (8010)
```bash
cd ~/ai_vision/subject13_ai_platform
source ~/ai_vision/subject10_pose_action/.venv/bin/activate
export SUBJECT10_ROOT="$HOME/ai_vision/subject10_pose_action"
export DAY04_STREAM_SOURCE='http://스마트폰IP:8080/video'
python -m uvicorn bridges.subject10_action:app --host 127.0.0.1 --port 8010
```

### 터미널 C — 교과 13 플랫폼 (8000)
```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
```

- 웹 페이지: http://127.0.0.1:8000/live
- 새 이벤트 목록: http://127.0.0.1:8000/api/v1/live/events
- DB 쿼리: `source_mode='live_inference'`
- 자동 검사: `python -m pytest -q`
- 카메라 URL, `.env`, 모델 파일·원본 영상은 공개 저장소에 업로드하지 않음
````

### 실습 24. Git에 추가할 파일을 확인합니다

기존 `.gitignore`에서 `.venv`, `.env`, `*.pt`, `*.joblib`, `*.onnx`, `*.mp4`, `handoff/records/`, `handoff/provided/` 등이 제외되어 있는지 확인합니다. 오늘 작성한 `.py`, `.html`, `.md` 소스코드는 업로드해도 되지만 카메라 URL·자격증명·가중치는 제외해야 합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git status
ls -lh   bridges/subject10_action.py   app/live_api.py app/live_repository.py   web/live.html tests/test_live.py   scripts/07_check_day04_inputs.py scripts/08_verify_live_records.py
```

---

# 6. 오늘 작성한 내용의 최종 정리 및 GitHub 업로드

## 6.1. 오늘 완성한 결과를 연결합니다

```text
1일차  기존 OCR·Action 결과 공통 JSON 조회
  ↓
2일차  PostgreSQL CRUD
  ↓
3일차  JSONL·Parquet·Gold + Redis·Celery Export
  ↓
4일차  실제 OCR 검사 + IP Webcam Pose/Action
  ↓
        공통 InferenceEvent
  ↓
        PostgreSQL 기록 저장·웹 조회
  ↓
5일차  Edge용 모델 준비·ONNX
```

오늘의 핵심은 **기존에 만들었던 AI 프로그램을 다시 실행하고, 결과를 저장·조회가 가능한 하나의 플랫폼에 연결**하는 것입니다. 과거 기록(`recorded_result`), 수동 실습(`manual_entry`), 실제 새 추론(`live_inference`)을 혼동하지 않고 유지했습니다.

## 6.2. 최종 자가 체크리스트

- [ ] 교과 9 OCR 원본 이미지 6장과 Rule/서비스 코드를 확인했습니다.
- [ ] 교과 10 Pose 가중치·RF 모델·학습정보·Feature 목록을 확인했습니다.
- [ ] OCR 서버 `8009`와 Action Bridge `8010`가 실행됩니다.
- [ ] `app/schemas.py`의 `source_mode`에 `live_inference`를 추가했습니다.
- [ ] 새 `app/live_repository.py`, `app/live_api.py` 코드를 작성했습니다.
- [ ] 기존 `app/main.py`에 4일차 Router만 추가했습니다.
- [ ] `/live`에서 실제 OCR 검사 결과를 확인하고 DB에 저장했습니다.
- [ ] `/live`에서 YOLO Pose BBox·Skeleton, Stable Label, FPS를 확인했습니다.
- [ ] 실제 Action 예측을 기준 동작과 비교해 DB에 저장했습니다.
- [ ] PostgreSQL에서 이전 기록과 새 추론 기록을 구분했습니다.
- [ ] 자동 테스트와 실제 카메라/모델 실행 점검을 각각 수행했습니다.
- [ ] 원본 데이터·가중치·개인 영상·접속 설정을 Git에 포함하지 않았습니다.

## 6.3. 4일차 Git 커밋을 남깁니다

새 저장소를 만들지 않습니다. **1~3일차와 같은 교과 13 프로젝트 저장소**에 오늘 기능을 추가합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git status
```

오늘 작성하거나 변경한 파일만 명시적으로 추가합니다.

```bash
git add   app/schemas.py app/main.py app/live_api.py app/live_repository.py   bridges/__init__.py bridges/subject10_action.py   web/live.html tests/test_live.py   scripts/07_check_day04_inputs.py scripts/08_verify_live_records.py   .env.example README.md docs/architecture.md

git status
```

`.env`, 모델 파일, 개인정보가 포함된 영상이 `git status`의 Staged 목록에 없는지 확인한 뒤 커밋합니다.

```bash
git commit -m "교과13 4일차: 실제 OCR·Pose Action 연동 및 PostgreSQL 저장"
git log -1 --oneline
git status
```

외부 GitHub를 사용할 수 있는 환경이라면 기존 원격에 푸시합니다.

```bash
git remote -v
git push origin main
```

폐쇄망에서 외부 GitHub가 금지되어 있다면 로컬 Git이나 허용된 내부 저장소만 사용합니다.

## 6.4. 5일차와 연결합니다

4일차까지는 개발용 PC에서 실제 AI 모델을 실행하고 결과를 데이터베이스·웹 화면으로 연결했습니다. **5일차에는 Raspberry Pi에서 실행하기 위한 경량 TinyCNN을 학습하여 ONNX로 변환**하고 Jetson 모델도 준비합니다. 지금 만든 공통 JSON·FastAPI·PostgreSQL을 버리지 않고, 새로운 Edge 장치가 보낸 결과를 같은 플랫폼에서 사용할 수 있도록 계속 확장합니다.

