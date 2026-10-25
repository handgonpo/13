## 1. 오늘의 목표

4일차에는 교과 9의 실제 OCR과 교과 10의 YOLO Pose·Action Recognition을 교과 13의 공통 플랫폼에 연결했습니다. AI 추론 결과를 웹 화면에서 보고 PostgreSQL에 저장할 수 있게 되었습니다.

그런데 모든 AI 모델을 Raspberry Pi처럼 연산 자원이 제한된 장치에서 같은 방식으로 실행할 수 있는 것은 아닙니다. 오늘은 **Raspberry Pi CPU에서 실행할 가벼운 이미지 분류 모델을 직접 만들고**, 개발 PC의 PyTorch 모델을 장치에서 재사용하기 좋은 ONNX 형식으로 내보냅니다. 한편 교과 10의 기존 YOLO11n-Pose는 Jetson에 배포할 모델 후보로 보존하고 ONNX 형식으로 준비합니다.

오늘 수업을 마치면 다음을 직접 수행할 수 있습니다.

- 4일차까지의 `subject13_ai_platform` 프로젝트를 그대로 다시 열고 이전 API를 유지합니다.
- `normal`과 `warning_red`의 교육용 이미지를 **Train / Validation / Test**로 분리하여 준비합니다.
- IP Webcam을 통해 직접 이미지를 촬영하거나, 제공받은 동일한 구조의 이미지를 활용합니다.
- 데이터 누락과 Split 간 완전 중복을 검사하고, 64×64 RGB 전처리 규격을 고정합니다.
- 작은 CNN인 `TinyCNN`을 CPU에서 학습하고 **Validation 성능 기준으로 모델을 선택**합니다.
- 학습된 PyTorch 모델을 ONNX로 내보내고 `onnx.checker`와 ONNX Runtime CPU로 검증합니다.
- 이전에 보지 않은 Test 이미지에서 PyTorch와 ONNX의 예측값이 일치하는지 확인합니다.
- 같은 교과 13 FastAPI에 ONNX 이미지 미리보기 API와 간단한 웹 화면을 추가합니다.
- Raspberry Pi 6일차용 모델·전처리·추론 코드를 한 폴더로 준비하고, 교과 10 YOLO Pose 모델을 Jetson용으로 내보냅니다.
- 자동 테스트, README, 모델 무결성 확인 및 Git 커밋을 완료합니다.

**오늘의 완성 기준:** `tinycnn_best.pt`와 `tinycnn.onnx`가 생성되고, 독립된 Test 이미지로 PyTorch–ONNX CPU 결과 일치 검증이 통과해야 합니다. `http://127.0.0.1:8000/api/v1/edge/page`에서 이미지를 선택하면 **실제 ONNX 모델의 결과**가 표시되고, `deploy/day05_raspberrypi/`에 다음 날 사용할 실행 파일이 준비되어야 합니다.

이 수업의 `warning_red`는 **화면에 빨간 표시 카드가 보이는 교육용 상황**입니다. 제조 불량이나 사람의 위험 행동을 판정하도록 검증된 모델은 아닙니다.

---

## 2. 오늘 수업의 전체 흐름

![[Pasted image 20261018130315.png]]

```text
[4일차까지 완성한 동일한 AI 플랫폼]
FastAPI :8000 ─ PostgreSQL ─ Redis/Celery ─ 교과9 OCR / 교과10 Pose·Action
                           │
                           ↓
                 5일차 프로젝트 다시 열기
                           ↓
          정상 장면 / 빨간 표시 카드 장면 정의
                           ↓
          IP Webcam 촬영 또는 제공 이미지 배치
                           ↓
             Train / Validation / Test 분리
                           ↓
                  이미지 파일 자동 검사
                           ↓
           TinyCNN 학습 (PyTorch / CPU / RGB)
                           ↓
              Validation으로 Best 모델 선택
                           ↓
                    tinycnn_best.pt
                           ↓
                ONNX 변환·구조 검사
                           ↓
                      tinycnn.onnx
                           ↓
              Test 이미지로 양쪽 결과 비교
                           ↓
            ONNX Runtime CPU 추론 규격 확정
                           ↓
        교과13 FastAPI /api/v1/edge/predict 추가
                           ↓
                웹에서 Test 이미지 조회
                           ↓
      Raspberry Pi용 모델·전처리·실행 코드 패키징
                           │
                           └─────────┐
                                     ↓
       [별도 교과10 .venv] YOLO11n-Pose → ONNX
                                     ↓
                         Jetson용 모델 후보 준비
                                     ↓
                   Pytest · README · Git 정리
                                     ↓
     [6일차] Raspberry Pi CPU 추론 + GPIO + 결과 전송
```

오늘은 **신규 모델의 학습 및 배포 준비**가 중심입니다. 4일차에 만든 실제 OCR·Action 서버를 지우거나 다시 작성하지 않습니다. PostgreSQL·Redis·Celery를 새로 설치하거나 Docker 컨테이너로 바꾸지도 않습니다.

---

## 3. 실습 전에 알아야 할 핵심

### 3.1. 기존 프로젝트에서 이어 사용할 파일

| 이전 일차 | 유지할 파일·기능 | 5일차와의 연결 |
|---|---|---|
| 1일차 | `app/main.py`, `app/schemas.py`, `app/records.py`, `web/index.html` | 같은 FastAPI 앱에 새 Router 연결 |
| 2일차 | `app/db.py`, `app/models.py`, `app/db_api.py` | 기존 PostgreSQL 서비스 유지. 오늘 모델 미리보기는 DB에 기록하지 않음 |
| 3일차 | `app/celery_app.py`, `app/pipeline_api.py`, `app/worker_tasks.py` | 비동기 Export 기능 유지. **학습이나 추론 속도를 높여 주는 기능이 아님** |
| 4일차 | `app/live_api.py`, `app/live_repository.py`, `web/live.html`, `bridges/subject10_action.py` | 실제 OCR·Pose 서비스와 별도로 5일차 Edge 모델 준비 |
| 교과 10 | `yolo11n-pose.pt`와 교과 10 `.venv` | Jetson용 Pose ONNX 후보 내보내기 |

**앞 교과의 결과물이 없는 경우:** 오늘 TinyCNN 실습을 위해 필요한 것은 `data/day05/images/{train,val,test}/{normal,warning_red}/`의 이미지 데이터입니다. 4일차까지 참여했다면 IP Webcam을 사용해 직접 수집할 수 있습니다. 촬영할 수 없다면 **같은 조건으로 준비한 교육용 이미지 폴더**를 받아 지정된 경로에 배치하세요. 교과 10의 `yolo11n-pose.pt`가 없다면 교과 10에서 사용한 신뢰할 수 있는 가중치 파일을 원래 프로젝트 위치에 배치해야 Jetson 모델 준비까지 진행할 수 있습니다. 이전 교과의 전체 GitHub ZIP을 다시 받는 작업은 하지 않습니다.

### 3.2. 오늘의 두 클래스는 어떤 뜻인가요?

| 클래스 | 촬영할 상황 | 수업에서의 해석 |
|---|---|---|
| `normal` (0) | 배경·사물이 보이지만 **빨간 표시 카드가 없는** 장면 | 교육용 정상 표시 |
| `warning_red` (1) | 같은 공간에서 **빨간 표시 카드가 뚜렷하게 보이는** 장면 | 교육용 주의 표시 |

이미지를 받은 위치나 파일명만 외우는 모델이 되지 않도록 카드를 놓는 위치, 카메라 거리, 배경, 조명을 바꾸어 찍습니다. **같은 사진을 Train·Validation·Test에 중복해서 넣지 않습니다.** 빨간 카드의 유무만을 학습한 모델이 다른 불량·위험 상황도 알 수 있다고 해석하면 안 됩니다.

### 3.3. Train · Validation · Test를 나누는 이유

| Split | 클래스별 준비량 | 목적 |
|---|---:|---|
| Train | 24장 | 가중치 학습 |
| Validation | 8장 | 학습 중 가장 적절한 체크포인트 선택 |
| Test | 8장 | 최종 모델과 ONNX 변환 결과를 **마지막에 한 번 검증** |
| 합계 | **각 클래스 40장, 총 80장** | 학생별 교육용 최소 실습 기준 |

서로 다른 촬영 세션 또는 조건에서 이미지를 준비하고, Test를 살펴보며 학습 설정을 계속 바꾸지 않습니다. **80장은 전체 실행 과정을 이해하기 위한 소규모 실습량이며 현업 품질을 보장하지 않습니다.** 이미지가 부족하다면 같은 사진을 단순 복사하지 말고 조건을 바꾸어 새로 촬영합니다.

### 3.4. TinyCNN·ONNX·ONNX Runtime은 무엇이 다른가요?

```text
PyTorch TinyCNN
→ CNN 구조 + 학습된 가중치 → tinycnn_best.pt (개발·학습용)
                         ↓
                     ONNX Export
                         ↓
                   tinycnn.onnx (추론 교환 형식)
                         ↓
            ONNX Runtime CPU (추론 실행 엔진)
                         ↓
       Raspberry Pi의 CPU 추론·GPIO 제어로 확장
```

`TinyCNN`은 작게 설계한 합성곱 신경망(CNN)입니다. `tinycnn_best.pt`에는 모델의 **가중치와 학습 메타데이터**를 저장하며, ONNX는 이 모델을 **장치 간 전달할 수 있는 추론 그래프**로 표현합니다. 오늘은 **이미 작은 CNN을 설계하는 방식으로 모델 크기를 줄여** 학습하며, INT8 양자화와 Jetson TensorRT 최적화는 수행하지 않습니다. ONNX 파일을 만든 것만으로 속도가 향상되는 것은 아닙니다. 실제 처리 속도는 장치·런타임·모델·입력 조건을 직접 측정해야 합니다.

### 3.5. 입력 전처리가 다르면 같은 모델도 다른 답을 냅니다

오늘은 다음 규격을 처음부터 끝까지 고정합니다.

```text
이미지 파일 → RGB → 64×64 Bilinear Resize
          → float32 / 255.0 (0~1)
          → NCHW Tensor [1, 3, 64, 64]
          → TinyCNN / ONNX 입력 이름 image
          → logits [1, 2] → Softmax → normal / warning_red
```

학습 때와 Raspberry Pi에서 RGB/BGR, 크기 또는 0~255/0~1 처리가 달라지면 결과가 변합니다. **`model.onnx`만 가져가지 않고 클래스 순서·전처리 설정·추론 코드까지 같이 가져가는 이유**입니다.

### 3.6. 교육용 OK·NG·UNKNOWN과 신뢰도

`normal`이 충분히 높은 신뢰도로 예측되면 `OK`, `warning_red`이면 `NG`로 표시합니다. 가장 높은 분류확률이 **0.60 미만**이면 `UNKNOWN`으로 판정을 유보합니다. `0.60`은 실습을 위한 임시값으로, 실제 제조·안전 운영에 검증된 Threshold가 아닙니다. 또한 CNN의 Softmax 확률을 곧바로 현업 수준의 검증된 신뢰도나 고장 확률로 해석하지 않습니다.

4일차의 OCR 검사에서 `confidence=null`을 보존한 규칙과도 다릅니다. **오늘 새 TinyCNN에서 계산한 확률만** 오늘의 미리보기 결과에 사용합니다.

### 3.7. Raspberry Pi와 Jetson에 준비할 모델은 서로 다릅니다

| 장치 | 오늘 준비하는 모델 | 다음 학습 연결 |
|---|---|---|
| Raspberry Pi | 직접 학습한 `TinyCNN → ONNX` | **6일차:** CPU 추론, 파랑/빨강 LED, 이벤트 전송 |
| Jetson Orin Nano Super | 교과 10의 `YOLO11n-Pose → ONNX` 후보 | **8~9일차:** ARM64 GPU 배포, TensorRT 변환 및 속도 비교 |

교과 10의 YOLO Pose를 오늘 TinyCNN 데이터로 재학습하지 않습니다. YOLO Pose와 TinyCNN은 **서로 다른 입력·출력과 용도**를 가진 모델입니다. Jetson의 TensorRT 엔진 파일은 오늘 PC에서 임의로 만들지 않고, 9일차에 실제 Jetson 환경에서 생성합니다.

---

## 4. 오늘 사용할 기술 스택

| 기술 | 오늘 실제로 하는 일 |
|---|---|
| Python `.venv` | 4일차까지 만든 교과 13 프로젝트에서 모델 학습 환경 추가 |
| OpenCV · IP Webcam | 스마트폰 Stream에서 80장 교육용 이미지 촬영(또는 제공 이미지 사용) |
| Pillow · NumPy | RGB 변환, 64×64 Resize, Tensor 입력 준비 |
| PyTorch | TinyCNN 구현, Train 학습, Validation 기반 Best 체크포인트 선택 |
| ONNX · ONNX Runtime | PyTorch 가중치 그래프 Export·검사·CPU 실행·동일성 확인 |
| FastAPI · HTML/JavaScript | 기존 플랫폼에 Edge 모델 미리보기 API·화면 추가 |
| Ultralytics YOLO · ONNX | 기존 교과 10 가상환경에서 Pose ONNX 후보 Export |
| Pytest · Git | 코드·데이터 전처리 규칙 자동검증, 수업 결과 기록 |

**실습 환경:** Windows 11 + WSL Ubuntu 또는 Ubuntu, 기존 VS Code·Git·Python 프로젝트를 사용합니다. 6일차 Raspberry Pi, 8일차 Jetson의 실제 배포는 **오늘 수행하지 않습니다.**

---

# 5. 처음부터 끝까지 따라 하는 상세 실습

## PART 1. 기존 프로젝트와 실행환경을 준비합니다

### 실습 1. 4일차 프로젝트를 다시 엽니다

WSL Ubuntu 터미널을 열고 다음을 차례대로 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
pwd
which python
python --version
git status --short
```

`pwd`의 끝에 `subject13_ai_platform`이 나오고, `which python`에는 현재 프로젝트의 `.venv/bin/python`이 표시되어야 합니다. **새 프로젝트와 새 가상환경을 만들지 않습니다.** `git status`에서 4일차 작업이 아직 커밋되지 않았다면 먼저 변경사항을 확인하세요.

이전 API 연결이 남아 있는지 확인합니다.

```bash
ls -lh app/main.py app/schemas.py app/db.py app/live_api.py
ls -lh app/pipeline_api.py bridges/subject10_action.py
ls -lh web/index.html web/db.html web/pipeline.html web/live.html
```

모두 존재하면 1~4일차 프로젝트를 올바르게 이어받았습니다. 일부 파일이 없으면 이전 일차의 해당 실습을 완료한 뒤 진행합니다.

### 실습 2. 5일차에 추가할 폴더를 만듭니다

```bash
mkdir -p \
  edge \
  scripts \
  data/day05/images/{train,val,test}/{normal,warning_red} \
  artifacts/day05/jetson \
  reports/day05 \
  deploy/day05_raspberrypi \
  docs tests web
```

프로젝트 구조는 기존 1~4일차 파일을 유지하며 다음 부분이 **추가**됩니다.

```text
subject13_ai_platform/
├── app/
│   ├── main.py                        [기존 파일: Router 1회 추가]
│   ├── db.py / models.py / live_api.py ... [그대로 유지]
│   └── edge_api.py                    [신규: ONNX 이미지 미리보기 API]
├── edge/                              [신규: Edge 모델 공통 모듈]
│   ├── __init__.py
│   ├── model.py
│   ├── dataset.py
│   ├── runtime.py
│   └── infer_on_device.py
├── scripts/
│   ├── 01~08_...                      [이전 일차 스크립트 유지]
│   ├── 09_capture_day05.py
│   ├── 10_check_day05_data.py
│   ├── 11_train_tinycnn.py
│   ├── 12_export_tinycnn_onnx.py
│   ├── 13_verify_onnx.py
│   ├── 14_package_day05.py
│   └── 15_prepare_jetson_pose.py
├── data/day05/images/                 [촬영한 개인별 이미지; Git 제외]
│   ├── train/normal/ · warning_red/
│   ├── val/normal/   · warning_red/
│   └── test/normal/  · warning_red/
├── artifacts/day05/                   [가중치, ONNX, 설정; Git 제외]
│   ├── tinycnn_best.pt
│   ├── training_config.json
│   ├── tinycnn.onnx
│   └── jetson/
├── deploy/day05_raspberrypi/          [6일차에 사용할 배포 폴더; Git 제외]
├── reports/day05/                     [학습·검증 기록; Git 제외]
├── web/edge.html                      [신규: Edge 모델 미리보기]
└── tests/test_day05.py                [신규 테스트]
```

위 목록은 완료 후의 구조입니다. 지금 생성한 폴더 안의 파일은 이어지는 실습에서 하나씩 작성합니다.

### 실습 3. 5일차 패키지를 기존 `requirements.txt`에 추가합니다

이전 1~4일차의 `requirements.txt`를 지우지 않고 **아래 패키지만 맨 아래에 추가**합니다. `opencv-python`이 이미 설치된 환경에서는 `opencv-python-headless`를 함께 중복 설치하지 않고 기존 OpenCV 패키지를 사용해도 됩니다.

**수정 파일: `requirements.txt` — 맨 아래에 추가**

```text
# 5일차 Edge AI / ONNX
numpy>=1.26,<3
Pillow>=10,<13
torch>=2.5,<3
onnx>=1.16,<2
onnxruntime>=1.18,<2
opencv-python-headless>=4.9,<5
python-multipart>=0.0.9,<1
```

오늘 TinyCNN 학습은 CPU만으로 충분합니다. 교과 13 가상환경에 PyTorch가 없다면 **CPU 빌드**를 먼저 설치할 수 있습니다. 교과 10의 `.venv`를 수정할 필요는 없습니다.

먼저 현재 가상환경의 PyTorch 설치 여부를 확인합니다.

```bash
python -m pip show torch
```

**PyTorch가 없을 때만** CPU 빌드를 다음 명령으로 설치합니다. 이미 호환되는 PyTorch가 있으면 이 명령을 생략하고 이어서 진행합니다.

```bash
python -m pip install 'torch>=2.5,<3' --index-url https://download.pytorch.org/whl/cpu
```

그다음 새로 추가된 나머지 패키지를 설치·확인합니다.

```bash
python -m pip install -r requirements.txt
python -m pip show torch onnx onnxruntime Pillow python-multipart
```

이미 호환되는 PyTorch가 있다면 CPU 빌드를 다시 설치할 필요는 없습니다. 교육장이 인터넷을 차단하는 경우에는 **승인된 내부 패키지 저장소나 미리 받은 호환 Wheel 파일**을 사용합니다. 인터넷이 없다는 이유로 이전 `.venv` 전체를 복사하지 않습니다.

설치를 확인합니다.

```bash
python - <<'PY'
import torch, onnx, onnxruntime, cv2, PIL
print('PyTorch:', torch.__version__)
print('ONNX:', onnx.__version__)
print('ONNX Runtime:', onnxruntime.__version__)
print('OpenCV:', cv2.__version__)
print('CPU 학습 가능:', True)
PY
```

이때 버전 숫자는 설치 시점에 따라 다를 수 있습니다. **핵심은 오류 없이 모든 모듈이 import되는 것**입니다.

---

## PART 2. 교육용 데이터를 직접 준비합니다

### 실습 4. Train · Validation · Test의 촬영 조건을 정합니다

스마트폰 IP Webcam을 4일차에 연결했던 것처럼 실행하고, 화면에 표시된 현재 Stream URL을 확인합니다.

```text
예시: http://192.168.0.10:8080/video
```

이 URL은 예시이며 실제 스마트폰 주소로 변경해야 합니다. **4일차 Pose Bridge가 같은 카메라를 사용 중이면 먼저 중지하고** 촬영합니다. 카메라에 사람의 얼굴이나 개인정보가 들어가지 않도록 배경을 구성합니다.

6번의 촬영 세션을 아래처럼 구분합니다.

| 세션 | 카메라 앞 상황 | 저장 위치 | 장수 |
|---|---|---|---:|
| A | 빨간 카드 없는 장면, 위치·거리 변화 | `train/normal/` | 24 |
| B | 빨간 카드 보이는 장면, 위치·거리 변화 | `train/warning_red/` | 24 |
| C | Train과 다른 조건의 일반 장면 | `val/normal/` | 8 |
| D | Train과 다른 조건의 빨간 카드 장면 | `val/warning_red/` | 8 |
| E | **최종 검증용** 새 일반 장면 | `test/normal/` | 8 |
| F | **최종 검증용** 새 빨간 카드 장면 | `test/warning_red/` | 8 |

Validation과 Test는 Train의 JPG를 복사해 만드는 것이 아니라, **새로 찍은 이미지**를 저장합니다. `Test` 장면은 이후의 학습 조건을 선택할 때 미리 확인하지 않습니다.

### 실습 5. IP Webcam 이미지 촬영 코드를 작성합니다

**이 코드는 무엇을 하나요?** OpenCV로 IP Webcam의 Frame을 읽어 일정 간격으로 JPG를 저장합니다. WSL에서 `cv2.imshow()` 창이 열리지 않는 환경도 있으므로 **파일 저장만** 진행합니다. 이미지가 저장된 후 VS Code 파일 탐색기로 직접 확인합니다.

**의사코드**

```text
source / split / label / count 입력
       ↓
카메라 또는 Stream 연결 확인
       ↓
기존 이미지가 있으면 덮어쓰지 않고 중지
       ↓
촬영 준비시간 대기
       ↓
Frame 읽기 → 320×240 JPG 저장 → 촬영 간격 대기
       ↓
지정된 이미지 개수 확인 → 카메라 종료
```

**파일: `scripts/09_capture_day05.py`**

```python
"""4일차 IP Webcam 주소로 5일차 이미지 데이터를 직접 수집합니다.

화면 표시 없이 촬영하므로 WSL에서 cv2.imshow()가 없어도 실행할 수 있습니다.
"""
import argparse
import time
from pathlib import Path

import cv2

ROOT = Path(__file__).resolve().parents[1]


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--source", required=True, help="IP Webcam /video URL, MP4 경로 또는 숫자 카메라 번호")
    parser.add_argument("--split", required=True, choices=["train", "val", "test"])
    parser.add_argument("--label", required=True, choices=["normal", "warning_red"])
    parser.add_argument("--count", type=int, default=24)
    parser.add_argument("--interval", type=float, default=0.7)
    parser.add_argument("--delay", type=float, default=5.0)
    args = parser.parse_args()
    if args.count < 1 or args.interval <= 0 or args.delay < 0:
        parser.error("count>=1, interval>0, delay>=0 이어야 합니다.")

    source = int(args.source) if args.source.isdecimal() else args.source
    cap = cv2.VideoCapture(source)
    if not cap.isOpened():
        raise RuntimeError(f"카메라/Stream을 열 수 없습니다: {args.source}")

    out_dir = ROOT / "data" / "day05" / "images" / args.split / args.label
    out_dir.mkdir(parents=True, exist_ok=True)
    if any(out_dir.glob("*.jpg")):
        cap.release()
        raise RuntimeError(f"기존 이미지가 있습니다. 덮어쓰기 방지를 위해 중지: {out_dir}")

    print(f"{args.delay:.1f}초 후 촬영합니다. label={args.label}, split={args.split}")
    time.sleep(args.delay)
    try:
        for index in range(1, args.count + 1):
            ok, frame = cap.read()
            if not ok or frame is None:
                raise RuntimeError(f"{index}번째 프레임 읽기 실패. 연결을 확인하세요.")
            image = cv2.resize(frame, (320, 240), interpolation=cv2.INTER_AREA)
            output = out_dir / f"{args.label}_{args.split}_{index:03d}.jpg"
            if not cv2.imwrite(str(output), image):
                raise OSError(f"이미지 저장 실패: {output}")
            print(f"[{index:02d}/{args.count}] {output.name}")
            if index < args.count:
                time.sleep(args.interval)
    finally:
        cap.release()
    print("완료. VS Code에서 촬영된 이미지를 확인하세요.")


if __name__ == "__main__":
    main()
```

프로젝트 루트에서 아래처럼 실행합니다. **현재 스마트폰 주소를 `SOURCE`에 설정한 뒤** 6개 명령을 순서대로 실행하세요.

```bash
SOURCE='http://스마트폰의실제IP:8080/video'

python -m scripts.09_capture_day05 --source "$SOURCE" --split train --label normal --count 24
python -m scripts.09_capture_day05 --source "$SOURCE" --split train --label warning_red --count 24

python -m scripts.09_capture_day05 --source "$SOURCE" --split val --label normal --count 8
python -m scripts.09_capture_day05 --source "$SOURCE" --split val --label warning_red --count 8

python -m scripts.09_capture_day05 --source "$SOURCE" --split test --label normal --count 8
python -m scripts.09_capture_day05 --source "$SOURCE" --split test --label warning_red --count 8
```

예를 들어 정상적인 첫 실행에서는 다음처럼 출력됩니다.

```text
5.0초 후 촬영합니다. label=normal, split=train
[01/24] normal_train_001.jpg
[02/24] normal_train_002.jpg
...
[24/24] normal_train_024.jpg
완료. VS Code에서 촬영된 이미지를 확인하세요.
```

**기존에 파일이 있어서 중지되면:** 촬영이 완료된 폴더인지 먼저 열어 확인하고, 다시 촬영해야 한다면 해당 Split·클래스 폴더의 이미지를 별도로 백업·정리한 뒤 실행합니다. 학생의 기존 이미지를 자동으로 삭제하지 않습니다.

**이미지를 제공받은 경우:** 촬영 명령은 실행하지 않고 받은 `train/`, `val/`, `test/`의 `normal/`, `warning_red/` 폴더를 `data/day05/images/` 아래에 그대로 배치합니다. 다른 일차의 OCR 이미지나 Pose 영상을 클래스 이름만 바꾸어 넣지 않습니다.

### 실습 6. 촬영한 데이터 수량과 파일을 확인합니다

```bash
find data/day05/images -type f \( -iname '*.jpg' -o -iname '*.png' \) | wc -l
find data/day05/images -type f -name '*.jpg' | sort | head -12
```

기본 구성이라면 총 **80장**입니다. 숫자가 맞더라도 클래스가 잘못 배치되거나 이미지가 손상되었을 수 있으므로 뒤의 데이터 자동 검사까지 실행해야 합니다.

---

## PART 3. TinyCNN 구조와 데이터 검사·학습 프로그램을 만듭니다

### 실습 7. 학습할 TinyCNN 모델의 구조를 작성합니다

**이번 코드는 무엇을 하나요?** 64×64 RGB 이미지에서 특징을 추출하고 `normal`, `warning_red` 두 클래스에 대한 점수(Logits)를 출력하는 가벼운 CNN을 만듭니다.

**의사코드**

```text
RGB 입력 [N,3,64,64]
       ↓
Conv + ReLU + Pool → Conv + ReLU + Pool
       ↓
Conv + ReLU + AdaptiveAvgPool
       ↓
Flatten + Linear(2)
       ↓
normal / warning_red의 Logits
```

먼저 Python 패키지 초기화 파일을 만듭니다.

**파일: `edge/__init__.py`**

```python
"""5일차 CPU TinyCNN → ONNX 학습 및 추론 도구."""
```

**파일: `edge/model.py`**

```python
"""TinyCNN 구조: 64×64 RGB 이미지를 두 종류로 분류합니다."""
import torch.nn as nn

CLASS_NAMES = ["normal", "warning_red"]  # 순서 고정 (0, 1)
IMAGE_SIZE = 64


class TinyCNN(nn.Module):
    def __init__(self) -> None:
        super().__init__()
        self.network = nn.Sequential(
            nn.Conv2d(3, 16, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.MaxPool2d(2),
            nn.Conv2d(16, 32, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.MaxPool2d(2),
            nn.Conv2d(32, 64, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.AdaptiveAvgPool2d((1, 1)),
            nn.Flatten(),
            nn.Linear(64, len(CLASS_NAMES)),
        )

    def forward(self, image):
        return self.network(image)  # logits: [batch, 2]
```

`CLASS_NAMES`의 순서는 **`normal=0`, `warning_red=1`**입니다. 학습, ONNX 변환, Raspberry Pi 추론에서 이 순서를 바꾸면 잘못된 판정이 발생할 수 있습니다. 모델의 출력은 Softmax 이전의 Logits이며, PyTorch의 `CrossEntropyLoss`가 내부적으로 알맞은 계산을 수행합니다.

모델 입력·출력 크기를 확인합니다.

```bash
python - <<'PY'
import torch
from edge.model import TinyCNN
with torch.inference_mode():
    out = TinyCNN().eval()(torch.zeros(1, 3, 64, 64))
print('모델 출력:', tuple(out.shape))
PY
```

예상 출력은 `모델 출력: (1, 2)`입니다.

### 실습 8. 데이터셋 읽기와 동일한 이미지 전처리를 작성합니다

**이번 코드는 무엇을 하나요?** Train/Validation/Test 폴더의 이미지와 라벨을 읽고, 공통 입력 규격인 RGB·64×64·float32/255.0·NCHW로 만듭니다. Train 데이터만 좌우 반전과 밝기 변경을 적용하고, Validation·Test에는 적용하지 않습니다.

**의사코드**

```text
train / val / test 하위에서 클래스 폴더 확인
              ↓
이미지 읽기 → RGB → 64×64 → float32 / 255
              ↓
Train: 가벼운 Augmentation / Val·Test: 원본 유지
              ↓
Tensor[3,64,64]와 클래스 번호 반환
```

**파일: `edge/dataset.py`**

```python
"""학습 이미지 로더. 클래스·Split과 색상 전처리 규칙을 고정합니다."""
import random
import hashlib
from pathlib import Path

import numpy as np
import torch
from PIL import Image, ImageEnhance
from torch.utils.data import Dataset

from edge.model import CLASS_NAMES, IMAGE_SIZE

EXTENSIONS = {".jpg", ".jpeg", ".png"}
MINIMUM = {"train": 24, "val": 8, "test": 8}


def image_to_tensor(image: Image.Image) -> torch.Tensor:
    """RGB → 64x64 → float32[0,1] → CHW. ONNX 입력과 동일합니다."""
    rgb = image.convert("RGB").resize(
        (IMAGE_SIZE, IMAGE_SIZE), Image.Resampling.BILINEAR
    )
    pixels = np.asarray(rgb, dtype=np.float32) / 255.0
    return torch.from_numpy(np.transpose(pixels, (2, 0, 1)).copy())


class ImageFolderDataset(Dataset):
    def __init__(self, root: Path, split: str, augment: bool = False) -> None:
        if split not in ("train", "val", "test"):
            raise ValueError("split은 train, val, test 중 하나여야 합니다.")
        self.samples: list[tuple[Path, int]] = []
        self.augment = augment
        for index, label in enumerate(CLASS_NAMES):
            folder = root / split / label
            if not folder.is_dir():
                raise FileNotFoundError(f"이미지 폴더가 없습니다: {folder}")
            paths = sorted(
                path for path in folder.iterdir()
                if path.is_file() and path.suffix.lower() in EXTENSIONS
            )
            for path in paths:
                self.samples.append((path, index))
        if not self.samples:
            raise ValueError(f"이미지가 없습니다: {root / split}")

    def __len__(self) -> int:
        return len(self.samples)

    def __getitem__(self, index: int) -> tuple[torch.Tensor, int]:
        path, label = self.samples[index]
        with Image.open(path) as file:
            image = file.convert("RGB")
        if self.augment:
            if random.random() < 0.5:
                image = image.transpose(Image.Transpose.FLIP_LEFT_RIGHT)
            image = ImageEnhance.Brightness(image).enhance(
                random.uniform(0.85, 1.15)
            )
        return image_to_tensor(image), label


def validate_dataset(root: Path, minimum: dict | None = None) -> dict:
    required = minimum or MINIMUM
    found: dict = {}
    split_digests: dict[str, set[str]] = {}
    for split in ("train", "val", "test"):
        hashes = set()
        found[split] = {}
        for label in CLASS_NAMES:
            folder = root / split / label
            if not folder.is_dir():
                raise FileNotFoundError(f"필수 폴더가 없습니다: {folder}")
            images = sorted(p for p in folder.iterdir() if p.suffix.lower() in EXTENSIONS and p.is_file())
            if len(images) < required[split]:
                raise ValueError(f"{split}/{label}: 최소 {required[split]}장, 현재 {len(images)}장")
            for path in images:
                with Image.open(path) as file:
                    file.verify()
                digest = hashlib.sha256(path.read_bytes()).hexdigest()
                if digest in hashes:
                    raise ValueError(f"{split} 안에 바이트가 동일한 이미지가 중복되었습니다: {path}")
                hashes.add(digest)
            found[split][label] = len(images)
        split_digests[split] = hashes

    for a, b in (("train", "val"), ("train", "test"), ("val", "test")):
        if split_digests[a] & split_digests[b]:
            raise ValueError(f"{a}와 {b}에 내용이 동일한 이미지 파일이 있습니다. 데이터 누수입니다.")
    return found
```

`image_to_tensor()`는 5일차에서 사용하는 **학습 입력의 기준**입니다. `ImageFolderDataset`은 실제 이미지 경로를 읽어 Train·Validation·Test를 분리하며, `validate_dataset()`은 파일이 열리는지, 클래스별 장수가 충분한지, **Split 사이에 동일한 파일이 섞였는지** 검사합니다.

자동 검사의 해시 비교는 바이트가 똑같은 파일을 찾는 방식입니다. **화면 내용이 거의 같은 연속 Frame이나 다시 JPEG로 저장한 복사본까지 완벽하게 찾아내지는 못하므로**, 촬영 세션을 구분하는 원칙도 지켜야 합니다.

### 실습 9. 데이터 자동 검사 코드를 작성합니다

**파일: `scripts/10_check_day05_data.py`**

```python
"""데이터 누락·이미지 오류·Split 간 중복을 확인합니다."""
import json
from pathlib import Path

from edge.dataset import validate_dataset

ROOT = Path(__file__).resolve().parents[1]
IMAGE_ROOT = ROOT / "data" / "day05" / "images"


def main() -> None:
    counts = validate_dataset(IMAGE_ROOT)
    output = ROOT / "reports" / "day05" / "dataset_inventory.json"
    output.parent.mkdir(parents=True, exist_ok=True)
    output.write_text(json.dumps(counts, indent=2, ensure_ascii=False), encoding="utf-8")
    print(json.dumps(counts, indent=2, ensure_ascii=False))
    print("Day05 dataset: READY")


if __name__ == "__main__":
    main()
```

실행합니다.

```bash
python -m scripts.10_check_day05_data
cat reports/day05/dataset_inventory.json
```

정상적인 기본 구성이라면 다음과 비슷하게 출력됩니다.

```json
{
  "train": {"normal": 24, "warning_red": 24},
  "val":   {"normal": 8,  "warning_red": 8},
  "test":  {"normal": 8,  "warning_red": 8}
}
```

마지막에 `Day05 dataset: READY`가 표시되면 학습으로 넘어갈 수 있습니다. 이미지가 부족하거나 열리지 않으면 오류에 나온 정확한 경로를 확인하고 다시 촬영하거나 파일을 복구합니다.

### 실습 10. TinyCNN 학습 코드를 작성합니다

**이번 코드는 무엇을 하나요?** `train`의 이미지만 이용해 CNN 가중치를 업데이트하고, **각 Epoch의 Validation Accuracy·Loss**를 비교하여 `tinycnn_best.pt`를 선택합니다. `test`의 이미지는 모델 선택에 사용하지 않습니다.

**의사코드**

```text
데이터 검사 통과
       ↓
Seed 고정 → Train·Validation DataLoader 생성
       ↓
TinyCNN CPU / CrossEntropyLoss / Adam
       ↓
Train 학습 → Validation 평가
       ↓
가장 좋은 Validation Accuracy·Loss의 가중치 저장
       ↓
학습 이력 JSON·입력 규격 설정 JSON 저장
       ↓
Test는 열지 않고 학습 종료
```

**파일: `scripts/11_train_tinycnn.py`**

```python
"""CPU TinyCNN 학습: Train으로 최적화, Validation으로 체크포인트 선택."""
import argparse
import json
import random
from pathlib import Path

import numpy as np
import torch
from torch import nn
from torch.utils.data import DataLoader

from edge.dataset import ImageFolderDataset, validate_dataset
from edge.model import CLASS_NAMES, IMAGE_SIZE, TinyCNN

ROOT = Path(__file__).resolve().parents[1]


def evaluate(model, loader, loss_fn):
    model.eval()
    correct, total, total_loss = 0, 0, 0.0
    with torch.inference_mode():
        for images, targets in loader:
            logits = model(images)
            total_loss += loss_fn(logits, targets).item() * len(targets)
            correct += (logits.argmax(dim=1) == targets).sum().item()
            total += len(targets)
    return total_loss / total, correct / total


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--epochs", type=int, default=12)
    parser.add_argument("--batch-size", type=int, default=8)
    parser.add_argument("--seed", type=int, default=42)
    args = parser.parse_args()
    if args.epochs < 1 or args.batch_size < 1:
        parser.error("epochs, batch-size는 1 이상이어야 합니다.")

    random.seed(args.seed)
    np.random.seed(args.seed)
    torch.manual_seed(args.seed)
    torch.set_num_threads(min(4, torch.get_num_threads()))
    data_dir = ROOT / "data" / "day05" / "images"
    counts = validate_dataset(data_dir)

    train_set = ImageFolderDataset(data_dir, "train", augment=True)
    val_set = ImageFolderDataset(data_dir, "val")
    train_loader = DataLoader(train_set, batch_size=args.batch_size, shuffle=True, num_workers=0)
    val_loader = DataLoader(val_set, batch_size=args.batch_size, shuffle=False, num_workers=0)

    model = TinyCNN().cpu()
    loss_fn = nn.CrossEntropyLoss()
    optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
    model_dir = ROOT / "artifacts" / "day05"
    report_dir = ROOT / "reports" / "day05"
    model_dir.mkdir(parents=True, exist_ok=True)
    report_dir.mkdir(parents=True, exist_ok=True)

    history = []
    best_acc, best_loss = -1.0, float("inf")
    for epoch in range(1, args.epochs + 1):
        model.train()
        train_total, train_correct, train_loss_sum = 0, 0, 0.0
        for images, targets in train_loader:
            optimizer.zero_grad(set_to_none=True)
            outputs = model(images)
            loss = loss_fn(outputs, targets)
            loss.backward()
            optimizer.step()
            train_loss_sum += loss.item() * len(targets)
            train_correct += (outputs.argmax(1) == targets).sum().item()
            train_total += len(targets)
        val_loss, val_acc = evaluate(model, val_loader, loss_fn)
        record = {
            "epoch": epoch,
            "train_loss": round(train_loss_sum / train_total, 5),
            "train_accuracy": round(train_correct / train_total, 5),
            "val_loss": round(val_loss, 5),
            "val_accuracy": round(val_acc, 5),
        }
        history.append(record)
        print(record)
        if (val_acc > best_acc) or (val_acc == best_acc and val_loss < best_loss):
            best_acc, best_loss = val_acc, val_loss
            torch.save(
                {"model_state_dict": model.state_dict(), "classes": CLASS_NAMES,
                 "image_size": IMAGE_SIZE, "epoch": epoch, "seed": args.seed},
                model_dir / "tinycnn_best.pt",
            )

    config = {
        "model_id": "tinycnn-day05", "model_version": "day05-1.0",
        "classes": CLASS_NAMES, "image_size": IMAGE_SIZE,
        "input_order": "RGB", "value_range": "0_to_1", "tensor_layout": "NCHW",
        "train_samples": counts["train"], "val_samples": counts["val"],
        "selected_by": "validation_accuracy_then_loss",
        "best_val_accuracy": best_acc, "best_epoch": torch.load(model_dir / "tinycnn_best.pt", map_location="cpu", weights_only=True)["epoch"],
        "seed": args.seed, "epochs_run": args.epochs,
        "confidence_threshold_demo": 0.60,
    }
    (model_dir / "training_config.json").write_text(json.dumps(config, indent=2), encoding="utf-8")
    (report_dir / "training_history.json").write_text(json.dumps(history, indent=2), encoding="utf-8")
    print(f"Saved: {model_dir / 'tinycnn_best.pt'}")
    print("Training finished. Test split은 아직 사용하지 않았습니다.")


if __name__ == "__main__":
    main()
```

프로젝트 루트에서 실행합니다.

```bash
python -m scripts.11_train_tinycnn --epochs 12 --batch-size 8
```

학습 시 Epoch마다 `train_loss`, `train_accuracy`, `val_loss`, `val_accuracy`가 출력됩니다. 숫자는 학생마다 촬영 이미지와 실행 환경에 따라 달라집니다. **특정 정확도를 반드시 얻는 것이 목적이 아니라 학습과 검증·모델 저장이 모두 연결되는지 확인하는 것이 목적**입니다.

학습이 완료되면 아래 파일이 생성됩니다.

```text
artifacts/day05/tinycnn_best.pt           ← PyTorch 체크포인트
artifacts/day05/training_config.json      ← RGB·64×64·클래스 순서·선택 Epoch
reports/day05/training_history.json       ← Epoch별 Train·Val 결과
```

```bash
ls -lh artifacts/day05/tinycnn_best.pt artifacts/day05/training_config.json
cat artifacts/day05/training_config.json
```

`training_config.json`에서 `classes`, `image_size`, `best_val_accuracy`, `best_epoch`을 확인합니다. 학습 데이터가 적기 때문에 Validation 점수가 높더라도 현업 성능으로 일반화하지 않습니다.

---

## PART 4. PyTorch 모델을 ONNX로 변환하고 독립된 Test에서 비교합니다

### 실습 11. PyTorch 체크포인트를 ONNX로 내보냅니다

**이번 코드는 무엇을 하나요?** 학습 때 선택한 `tinycnn_best.pt`를 읽어 같은 `TinyCNN` 구조에 가중치를 넣고, ONNX 추론 그래프인 `tinycnn.onnx`로 저장합니다. 고정 입력은 `image [1,3,64,64]`, 출력은 `logits [1,2]`입니다.

**의사코드**

```text
체크포인트/학습 설정 읽기
       ↓
클래스 순서와 이미지 크기 확인
       ↓
TinyCNN에 State Dict 로드 → eval()
       ↓
CPU Dummy Tensor [1,3,64,64] 준비
       ↓
ONNX Opset 17로 Export
       ↓
ONNX Graph 구조 검사 → 저장 확인
```

**파일: `scripts/12_export_tinycnn_onnx.py`**

```python
"""검증 단계에서 선택한 TinyCNN 체크포인트를 고정 배치 ONNX로 변환합니다."""
import json
from pathlib import Path

import onnx
import torch

from edge.model import CLASS_NAMES, IMAGE_SIZE, TinyCNN

ROOT = Path(__file__).resolve().parents[1]
ARTIFACT = ROOT / "artifacts" / "day05"


def main() -> None:
    state_file = ARTIFACT / "tinycnn_best.pt"
    config_file = ARTIFACT / "training_config.json"
    if not state_file.is_file() or not config_file.is_file():
        raise FileNotFoundError("11_train_tinycnn.py부터 실행하세요.")
    checkpoint = torch.load(state_file, map_location="cpu", weights_only=True)
    config = json.loads(config_file.read_text(encoding="utf-8"))
    if checkpoint["classes"] != CLASS_NAMES or config["classes"] != CLASS_NAMES:
        raise ValueError("클래스 순서가 현재 코드와 학습 체크포인트에서 다릅니다.")
    if checkpoint["image_size"] != IMAGE_SIZE:
        raise ValueError("모델 입력 크기가 다릅니다.")

    model = TinyCNN().cpu().eval()
    model.load_state_dict(checkpoint["model_state_dict"])
    dummy = torch.zeros(1, 3, IMAGE_SIZE, IMAGE_SIZE, dtype=torch.float32)
    target = ARTIFACT / "tinycnn.onnx"
    torch.onnx.export(
        model, dummy, str(target), export_params=True, opset_version=17,
        input_names=["image"], output_names=["logits"], dynamo=False,
    )
    graph = onnx.load(str(target))
    onnx.checker.check_model(graph)
    print("ONNX export: PASS")
    print("입력: image [1,3,64,64], float32 / 출력: logits [1,2]")
    print("저장:", target)


if __name__ == "__main__":
    main()
```

실행합니다.

```bash
python -m scripts.12_export_tinycnn_onnx
ls -lh artifacts/day05/tinycnn.onnx
```

`ONNX export: PASS`가 표시되고 ONNX 파일이 생성되면 다음 단계로 진행합니다. 일부 PyTorch 버전에서는 ONNX 내보내기 과정의 경고가 출력될 수 있습니다. **경고인지 예외인지 확인하고**, 예외로 중단되면 PyTorch·ONNX 설치 버전과 `onnx` 패키지 존재 여부를 점검합니다.

### 실습 12. ONNX Runtime의 CPU 추론 코드를 작성합니다

**이번 코드는 무엇을 하나요?** Raspberry Pi에서도 재사용할 수 있도록 PyTorch를 설치하지 않고 **ONNX Runtime + Pillow + NumPy**만으로 이미지 전처리와 예측을 수행합니다.

**의사코드**

```text
이미지 bytes → PIL RGB → [1,3,64,64] float32
       ↓
ONNX Runtime CPUInferenceSession 실행
       ↓
logits[1,2] → Softmax
       ↓
normal 또는 warning_red + 예측 확률
       ↓
Confidence<0.60 → UNKNOWN
그 외 normal → OK / warning_red → NG
       ↓
결과 JSON 반환
```

**파일: `edge/runtime.py`**

```python
"""Raspberry Pi에서도 재사용할 ONNX Runtime CPU 추론 함수."""
import io
import json
import time
from functools import lru_cache
from pathlib import Path

import numpy as np
import onnxruntime as ort
from PIL import Image


@lru_cache(maxsize=2)
def get_session(model_path: str) -> ort.InferenceSession:
    return ort.InferenceSession(model_path, providers=["CPUExecutionProvider"])


def preprocess_image(image: Image.Image, size: int) -> np.ndarray:
    rgb = image.convert("RGB").resize((size, size), Image.Resampling.BILINEAR)
    x = np.asarray(rgb, dtype=np.float32) / 255.0
    return np.transpose(x, (2, 0, 1))[None, :, :, :].copy()


def predict_image_bytes(
    content: bytes,
    model_path: Path,
    config_path: Path,
) -> dict:
    if not model_path.is_file() or not config_path.is_file():
        raise FileNotFoundError("ONNX 또는 training_config.json이 없습니다.")
    config = json.loads(config_path.read_text(encoding="utf-8"))
    classes = config["classes"]
    if classes != ["normal", "warning_red"]:
        raise ValueError("2개 클래스의 순서가 올바르지 않습니다.")
    with Image.open(io.BytesIO(content)) as original:
        x = preprocess_image(original, int(config["image_size"]))
    session = get_session(str(model_path.resolve()))
    if session.get_inputs()[0].name != "image":
        raise ValueError("ONNX 입력 이름(image)이 예상과 다릅니다.")
    start = time.perf_counter()
    logits = session.run(["logits"], {"image": x})[0][0]
    elapsed_ms = (time.perf_counter() - start) * 1000.0
    shifted = logits - np.max(logits)
    p = np.exp(shifted)
    probs = p / np.sum(p)
    index = int(np.argmax(probs))
    label = classes[index]
    confidence = float(probs[index])
    threshold = float(config["confidence_threshold_demo"])
    decision = "UNKNOWN" if confidence < threshold else ("OK" if label == "normal" else "NG")
    return {
        "predicted_label": label,
        "confidence": round(confidence, 6),
        "decision": decision,
        "probabilities": {name: round(float(probs[i]), 6) for i, name in enumerate(classes)},
        "inference_ms": round(elapsed_ms, 3),
        "model_id": config["model_id"],
        "model_version": config["model_version"],
        "source_mode": "edge_training_demo",
    }
```

이 코드는 **모델이 학습한 클래스의 의미를 바꾸지 않습니다.** `confidence_threshold_demo=0.60`은 `training_config.json`에서 읽고, 직접 만든 교육용 이미지 분류에만 적용합니다. `inference_ms`는 해당 ONNX Runtime Session의 **한 회 추론 시간**이며, 카메라 캡처·이미지 전송·웹 화면 렌더링 시간을 포함한 전체 FPS와는 다릅니다.

### 실습 13. 마지막 Test 이미지에서 두 모델의 결과를 비교합니다

학습을 끝낸 뒤에도 **ONNX 변환 과정에서 예측이 바뀌지 않았는지** 확인해야 합니다. 이를 위해 Train·Validation에서 사용하지 않은 Test 이미지들을 입력하여 PyTorch의 Logits와 ONNX Runtime의 Logits를 비교합니다.

**의사코드**

```text
Best 체크포인트 / ONNX 모델 불러오기
       ↓
Test 이미지 한 장씩 읽기 (학습 재실행 없음)
       ↓
동일한 RGB/Resize/Scale 전처리 적용
       ↓
PyTorch 출력 vs ONNX Runtime 출력 비교
       ↓
최대 Logit 차이 · 예측 일치율 · 각각의 Test 정확도
       ↓
ONNX CPU 추론 시간 p50 계산
       ↓
검증 보고서 JSON 저장
```

**파일: `scripts/13_verify_onnx.py`**

```python
"""별도 Test 데이터로 PyTorch–ONNX 결과 일치·교육용 성능을 확인합니다."""
import io
import json
import time
from pathlib import Path

import numpy as np
import onnxruntime as ort
import torch
from PIL import Image

from edge.dataset import ImageFolderDataset, image_to_tensor
from edge.model import CLASS_NAMES, TinyCNN
from edge.runtime import preprocess_image

ROOT = Path(__file__).resolve().parents[1]
ARTIFACT = ROOT / "artifacts" / "day05"
DATA = ROOT / "data" / "day05" / "images"


def main() -> None:
    checkpoint = torch.load(ARTIFACT / "tinycnn_best.pt", weights_only=True, map_location="cpu")
    model = TinyCNN().cpu().eval()
    model.load_state_dict(checkpoint["model_state_dict"])
    session = ort.InferenceSession(str(ARTIFACT / "tinycnn.onnx"), providers=["CPUExecutionProvider"])
    test_set = ImageFolderDataset(DATA, "test")
    matches = 0
    correct_torch, correct_onnx = 0, 0
    differences, milliseconds, rows = [], [], []
    with torch.inference_mode():
        for path, label in test_set.samples:
            content = path.read_bytes()
            with Image.open(io.BytesIO(content)) as file:
                image = file.convert("RGB")
            torch_logits = model(image_to_tensor(image)[None]).numpy()[0]
            x = preprocess_image(image, 64)
            start = time.perf_counter()
            onnx_logits = session.run(["logits"], {"image": x})[0][0]
            milliseconds.append((time.perf_counter() - start) * 1000)
            delta = float(np.max(np.abs(torch_logits - onnx_logits)))
            differences.append(delta)
            pt_idx = int(torch_logits.argmax())
            ox_idx = int(onnx_logits.argmax())
            matches += pt_idx == ox_idx
            correct_torch += pt_idx == label
            correct_onnx += ox_idx == label
            rows.append({"file": path.name, "label": CLASS_NAMES[label],
                         "torch_prediction": CLASS_NAMES[pt_idx],
                         "onnx_prediction": CLASS_NAMES[ox_idx],
                         "max_abs_logit_error": delta})
    n = len(test_set)
    report = {
        "test_samples": n,
        "pytorch_test_accuracy": round(correct_torch / n, 4),
        "onnx_test_accuracy": round(correct_onnx / n, 4),
        "prediction_agreement": round(matches / n, 4),
        "max_abs_logit_error": max(differences),
        "onnx_cpu_inference_p50_ms": round(float(np.median(milliseconds)), 3),
        "results": rows,
    }
    target = ROOT / "reports" / "day05" / "onnx_verification.json"
    target.parent.mkdir(parents=True, exist_ok=True)
    target.write_text(json.dumps(report, indent=2, ensure_ascii=False), encoding="utf-8")
    print(json.dumps({k: v for k, v in report.items() if k != "results"}, indent=2))
    if matches != n or max(differences) > 0.005:
        raise AssertionError("PyTorch/ONNX 결과 차이가 큽니다. 입력/가중치/ONNX 변환을 점검하세요.")
    print("PyTorch / ONNX CPU 일치 검증: PASS")
    print("참고: 교육용 소규모 Test 성능은 산업현장 일반화 성능이 아닙니다.")


if __name__ == "__main__":
    main()
```

실행합니다.

```bash
python -m scripts.13_verify_onnx
cat reports/day05/onnx_verification.json
```

출력에서 `prediction_agreement`가 `1.0`이고 `max_abs_logit_error`가 `0.005` 이하여야 현재의 ONNX Export 결과를 **일치 검증 통과**로 처리합니다. `onnx_test_accuracy`는 실제 입력 이미지에 따라 달라지며 **100%라고 예상하거나 보장하지 않습니다.**

한쪽 결과가 다르다면 RGB/BGR, Resize 방식, `IMAGE_SIZE`, 가중치 버전, ONNX Runtime 입력 이름과 출력 순서를 차례대로 확인합니다. **Test 결과를 좋게 만들기 위해 학습 설정을 반복 변경하지 않습니다.**

---

## PART 5. 완성한 교과 13 FastAPI에 ONNX 미리보기 기능을 추가합니다

### 실습 14. ONNX 이미지 업로드 API를 작성합니다

4일차의 OCR·Pose API와 2일차 DB API를 그대로 두고, 같은 FastAPI에 **5일차 Edge 모델의 확인용 API**를 추가합니다.

```text
POST /api/v1/edge/predict
→ 브라우저에서 1개 이미지 업로드
→ 5일차 TinyCNN ONNX Runtime CPU 추론
→ predicted_label · confidence · decision · inference_ms 반환

GET /api/v1/edge/health
→ ONNX와 학습 설정 파일의 존재 여부 확인

GET /api/v1/edge/page
→ 미리보기 웹 페이지
```

**이번 API는 모델 정상동작을 확인하는 미리보기입니다.** 오늘은 자동으로 PostgreSQL 이벤트를 저장하지 않으며, 6일차에 Raspberry Pi 장치가 만든 추론 결과를 기존 플랫폼에 전송하도록 확장합니다.

**파일: `app/edge_api.py`**

```python
"""5일차 ONNX 교육용 이미지 미리보기. 기존 교과 13 라우터를 유지합니다."""
from pathlib import Path

from fastapi import APIRouter, File, HTTPException, UploadFile
from fastapi.responses import FileResponse

from edge.runtime import predict_image_bytes

router = APIRouter(prefix="/api/v1/edge", tags=["day05-edge"])
ROOT = Path(__file__).resolve().parents[1]
ARTIFACT = ROOT / "artifacts" / "day05"
MODEL = ARTIFACT / "tinycnn.onnx"
CONFIG = ARTIFACT / "training_config.json"
PAGE = ROOT / "web" / "edge.html"


@router.get("/health")
def edge_health():
    return {"model_ready": MODEL.is_file() and CONFIG.is_file(), "mode": "education_demo"}


@router.post("/predict")
async def edge_predict(file: UploadFile = File(...)):
    data = await file.read(5 * 1024 * 1024 + 1)
    if len(data) > 5 * 1024 * 1024:
        raise HTTPException(status_code=413, detail="최대 5MB 이미지까지 허용합니다.")
    if not MODEL.is_file() or not CONFIG.is_file():
        raise HTTPException(status_code=503, detail="ONNX/설정 파일이 없습니다. 5일차 변환을 먼저 완료하세요.")
    try:
        return predict_image_bytes(data, MODEL, CONFIG)
    except (OSError, ValueError, KeyError) as exc:
        raise HTTPException(status_code=422, detail=f"올바른 이미지/모델 설정인지 확인하세요: {exc}") from exc


@router.get("/page", include_in_schema=False)
def edge_page():
    return FileResponse(PAGE)
```

여기에서는 이미지 본문을 최대 5MB로 제한하고, ONNX 파일이 준비되지 않았으면 HTTP 503을 반환합니다. 카메라 영상을 업로드하거나 개인 식별 이미지를 저장하는 기능은 만들지 않습니다.

### 실습 15. 기존 `app/main.py`에 5일차 Router를 연결합니다

**수정 파일: `app/main.py` — 맨 아래에 두 줄만 추가**

```python
# 2일차 PostgreSQL Router — 기존 그대로 유지
from app.db_api import router as db_router
app.include_router(db_router)

# 3일차 Redis·Celery Router — 기존 그대로 유지
from app.pipeline_api import router as pipeline_router
app.include_router(pipeline_router)

# 4일차 OCR·Pose Router — 기존 그대로 유지
from app.live_api import router as live_router
app.include_router(live_router)

# 5일차 ONNX 미리보기 Router — 이번에 두 줄만 신규 추가
from app.edge_api import router as edge_router
app.include_router(edge_router)
```

위 코드는 **기존 Router와 이번 신규 Router의 최종 모습**입니다. 이미 작성된 2~4일차 코드를 중복해서 붙여 넣지 말고, **5일차 두 줄만 한 번 추가**해야 합니다. `app = FastAPI(...)`를 또 작성하지 않습니다.

### 실습 16. ONNX 결과를 표시할 웹 화면을 작성합니다

**이번 코드는 무엇을 하나요?** 학습에서 사용하지 않은 Test 이미지를 선택하여 FastAPI의 `/api/v1/edge/predict`로 전송하고, ONNX Runtime CPU의 실제 결과 JSON을 화면에 표시합니다.

**파일: `web/edge.html`**

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>5일차 · Edge TinyCNN ONNX</title>
  <style>
    body{font-family:system-ui,sans-serif;max-width:760px;margin:32px auto;padding:0 14px;color:#203040}
    section{border:1px solid #cbd5df;border-radius:12px;padding:20px;margin-top:14px}
    button,input{font:inherit;padding:8px}pre{white-space:pre-wrap;overflow-wrap:anywhere;background:#f4f6fa;padding:15px}
    small{color:#667085}
  </style>
</head>
<body>
  <h1>교과 13 · 5일차 ONNX 추론 확인</h1>
  <p>학습한 TinyCNN을 ONNX Runtime CPU로 실행합니다. 기존 교과 9·10의 AI 모델과 별도의 교육용 분류기입니다.</p>
  <section>
    <label for="photo">Test 이미지 선택</label>
    <input id="photo" type="file" accept="image/*">
    <button id="run">추론 실행</button>
    <p><small>기존 PostgreSQL에는 저장하지 않습니다. 6일차에 Raspberry Pi 추론 결과를 연결합니다.</small></p>
  </section>
  <section><h2>예측 결과</h2><pre id="result">이미지를 선택하고 실행하세요.</pre></section>
  <script>
    const button=document.getElementById('run');
    const output=document.getElementById('result');
    button.addEventListener('click', async()=>{
      const photo=document.getElementById('photo').files[0];
      if(!photo){output.textContent='이미지를 먼저 선택하세요.';return;}
      const form=new FormData();form.append('file',photo);
      try{
        const response=await fetch('/api/v1/edge/predict',{method:'POST',body:form});
        const result=await response.json();
        if(!response.ok)throw new Error(JSON.stringify(result));
        output.textContent=JSON.stringify(result,null,2);
      }catch(error){output.textContent=`오류: ${error.message}`;}
    });
  </script>
</body>
</html>
```

### 실습 17. 같은 교과 13 서버에서 실제 ONNX 이미지를 확인합니다

교과 13 프로젝트의 `.venv`를 활성화한 터미널에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

서버가 실행되면 브라우저에서 다음 주소를 엽니다.

```text
http://127.0.0.1:8000/api/v1/edge/page
http://127.0.0.1:8000/api/v1/edge/health
http://127.0.0.1:8000/docs
```

`/api/v1/edge/health`에서 `model_ready: true`가 나오는지 확인한 뒤 웹 화면에서 `data/day05/images/test/normal/` 또는 `test/warning_red/`의 이미지를 한 장 선택하고 **추론 실행**을 누릅니다. 결과에는 `predicted_label`, `confidence`, `decision`, `inference_ms`, `source_mode: edge_training_demo`가 표시됩니다.

`source_mode=edge_training_demo`는 **이번 웹 미리보기 응답의 구분값**입니다. 1~4일차의 PostgreSQL `InferenceEvent.source_mode`에 추가하여 저장한 값이 아닙니다. 따라서 기존 DB나 `InferenceEvent`에 변경을 가하지 않습니다.

명령으로도 업로드할 수 있습니다. 서버를 유지한 채 새로운 터미널을 열어 실행합니다.

```bash
curl -X POST 'http://127.0.0.1:8000/api/v1/edge/predict' \
  -F 'file=@data/day05/images/test/normal/normal_test_001.jpg'
```

직접 촬영하지 않고 교재용 이미지를 배치하여 파일명이 다른 경우에는 실제 테스트 이미지 경로로 바꿔 실행합니다. **오늘 새 모델이 정상 판정했는지 여부와 무관하게 실제 출력값을 기록**하고, 판정이 어긋난 경우 원인을 관찰합니다.

---

## PART 6. Jetson에서 사용할 교과 10의 YOLO Pose 모델 후보를 준비합니다

### 실습 18. 교과 10에서 사용한 모델 파일을 확인합니다

6일차 Raspberry Pi에는 방금 학습한 TinyCNN을 사용합니다. 그러나 8~9일차 Jetson 실습에서는 **교과 10의 실제 YOLO11n-Pose**를 그대로 활용합니다. 둘은 같은 ONNX 형식이라도 출력의 의미가 다르므로 한 모델로 합치지 않습니다.

먼저 기존 교과 10 모델이 있는지 확인합니다.

```bash
ls -lh ~/ai_vision/subject10_pose_action/yolo11n-pose.pt
ls -lh ~/ai_vision/subject10_pose_action/.venv/bin/python
```

결석 또는 재설치로 가중치 파일이 없다면 교과 10 수업 당시 사용한 가중치를 원래 경로에 배치합니다. **신뢰할 수 없는 출처의 `.pt`나 `.joblib` 파일을 실행하지 않습니다.**

### 실습 19. 교과 10 환경에서 YOLO Pose를 ONNX로 내보냅니다

교과 10은 Ultralytics와 PyTorch 등을 이미 사용하는 프로젝트입니다. 기존 교과 10 가상환경의 라이브러리로 Export하고, 생성 파일은 **교과 13의 `artifacts/day05/jetson/`에만 저장**합니다. 원본 `yolo11n-pose.pt`는 수정하지 않습니다.

**의사코드**

```text
교과 10의 yolo11n-pose.pt 존재 확인
       ↓
교과 13 artifacts/day05/jetson/으로 복사
       ↓
Ultralytics YOLO 모델 로드 → CPU로 ONNX Export
       ↓
ONNX 구조 검사 → PT/ONNX 파일 크기 기록
       ↓
SHA256과 모델 정보 Manifest 저장
       ↓
8~9일차 Jetson 배포용 후보 확정
```

**파일: `scripts/15_prepare_jetson_pose.py`**

```python
"""교과 10 YOLO11n-Pose 가중치를 복사하여 Jetson용 ONNX 후보를 내보냅니다.

이 스크립트는 교과 10의 .venv에서 실행합니다(ultralytics, onnx 필요).
원본 교과 10 파일은 수정하지 않습니다.
"""
import argparse
import hashlib
import json
import shutil
from pathlib import Path

import onnx
from ultralytics import YOLO

ROOT = Path(__file__).resolve().parents[1]
TARGET = ROOT / "artifacts" / "day05" / "jetson"


def sha256(path: Path) -> str:
    return hashlib.sha256(path.read_bytes()).hexdigest()


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--source-pt", required=True, type=Path)
    args = parser.parse_args()
    original = args.source_pt.expanduser().resolve()
    if not original.is_file():
        raise FileNotFoundError(f"교과 10 YOLO Pose 가중치가 없습니다: {original}")
    TARGET.mkdir(parents=True, exist_ok=True)
    copied = TARGET / "yolo11n-pose.pt"
    shutil.copy2(original, copied)
    exported = Path(YOLO(str(copied)).export(
        format="onnx", imgsz=640, opset=17, simplify=False,
        batch=1, device="cpu",
    ))
    if not exported.is_file():
        raise RuntimeError("YOLO ONNX 파일이 생성되지 않았습니다.")
    onnx.checker.check_model(onnx.load(str(exported)))
    report = {
        "model": "YOLO11n-Pose (교과10 재사용)",
        "source_pt": str(original),
        "copied_pt": copied.name,
        "exported_onnx": exported.name,
        "imgsz": 640, "batch": 1, "opset": 17,
        "pt_bytes": copied.stat().st_size,
        "onnx_bytes": exported.stat().st_size,
        "onnx_sha256": sha256(exported),
        "note": "Jetson TensorRT 엔진은 9일차에 Jetson에서 생성·측정",
    }
    (TARGET / "pose_export_manifest.json").write_text(
        json.dumps(report, indent=2, ensure_ascii=False), encoding="utf-8"
    )
    print("Jetson용 YOLO Pose ONNX 후보: PASS")
    print("원본 PT:", original)
    print("복사한 PT:", copied)
    print("내보낸 ONNX:", exported)


if __name__ == "__main__":
    main()
```

교과 13 프로젝트 **최상위 폴더에 있는 상태에서**, 교과 10 `.venv`의 Python을 직접 지정해 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform

# 교과 10 가상환경에 ONNX 패키지가 준비되어 있는지 확인합니다.
~/ai_vision/subject10_pose_action/.venv/bin/python -m pip show ultralytics onnx

# 패키지가 없다면 승인된 저장소에서 교과 10 가상환경에만 설치합니다.
~/ai_vision/subject10_pose_action/.venv/bin/python -m pip install 'onnx>=1.16,<2'

# 교과 10의 실제 YOLO Pose를 ONNX 후보로 내보냅니다.
~/ai_vision/subject10_pose_action/.venv/bin/python scripts/15_prepare_jetson_pose.py \
  --source-pt ~/ai_vision/subject10_pose_action/yolo11n-pose.pt
```

실행을 마치면 `artifacts/day05/jetson/yolo11n-pose.onnx`와 `pose_export_manifest.json` 등이 생성됩니다. Ultralytics Export 과정에서 교과 10 환경과 ONNX 버전의 호환성 오류가 발생하면 **실행 메시지와 사용 중인 패키지 버전을 먼저 확인**합니다. 9일차의 TensorRT 성능 비교를 아직 수행한 것은 아닙니다.

확인합니다.

```bash
ls -lh artifacts/day05/jetson/
cat artifacts/day05/jetson/pose_export_manifest.json
```

Manifest에서 PT·ONNX 크기를 비교하고, ONNX SHA256을 확인합니다. **파일 크기가 작다고 반드시 FPS가 더 빠른 것은 아닙니다.** 모델 추론의 품질과 속도는 실제 Jetson에서 측정해야 합니다.

---

## PART 7. Raspberry Pi에 가져갈 실행 파일을 준비합니다

### 실습 20. Raspberry Pi용 단독 추론 명령을 작성합니다

**이번 코드는 무엇을 하나요?** 6일차 Raspberry Pi에서 이미지 파일 한 장을 ONNX 모델로 분석하고, `predicted_label`, `confidence`, `decision`을 JSON으로 출력합니다. 현장 카메라와 GPIO 제어는 **6일차에 추가**하므로 오늘은 파일 입력 단계까지만 준비합니다.

**파일: `edge/infer_on_device.py`**

```python
"""6일차 Raspberry Pi에서 직접 실행할 추론 CLI. 배포 폴더에서 실행합니다."""
import argparse
import json
from pathlib import Path

from runtime import predict_image_bytes


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--image", required=True, type=Path)
    args = parser.parse_args()
    here = Path(__file__).resolve().parent
    image_path = args.image.expanduser().resolve()
    if not image_path.is_file():
        raise FileNotFoundError(image_path)
    result = predict_image_bytes(
        image_path.read_bytes(),
        here / "tinycnn.onnx",
        here / "training_config.json",
    )
    result["input_ref"] = image_path.name
    print(json.dumps(result, indent=2, ensure_ascii=False))


if __name__ == "__main__":
    main()
```

이 파일은 원래 개발 프로젝트의 `edge/` 폴더에 작성하지만, **배포할 때는 `runtime.py`와 같은 폴더로 복사하여 실행**합니다. 직접 `python -m edge.infer_on_device`로 실행하지 않고, 아래 패키징 단계를 마친 후 배포 폴더에서 실행합니다.

### 실습 21. ONNX·전처리·Manifest를 배포 폴더로 모읍니다

**이번 코드는 무엇을 하나요?** ONNX 모델만 복사하지 않고, 학습 시 고정한 클래스 순서·전처리 규칙·교육용 Threshold·추론 코드를 함께 모읍니다. 또한 SHA256을 계산하여 6일차에 파일이 변하지 않았는지 확인할 수 있도록 합니다.

**의사코드**

```text
tinycnn.onnx / training_config.json / ONNX 검증 보고서 확인
       ↓
Raspberry Pi 전용 폴더 생성
       ↓
모델·설정·runtime.py·infer_on_device.py 복사
       ↓
모델 SHA256 계산
       ↓
manifest.json 작성
       ↓
배포 대상 확인
```

**파일: `scripts/14_package_day05.py`**

```python
"""6일차 Raspberry Pi 이관용 ONNX·규격·추론 스크립트를 준비합니다."""
import hashlib
import json
import shutil
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]
ARTIFACT = ROOT / "artifacts" / "day05"
OUT = ROOT / "deploy" / "day05_raspberrypi"


def main() -> None:
    needed = [ARTIFACT / "tinycnn.onnx", ARTIFACT / "training_config.json",
              ROOT / "reports" / "day05" / "onnx_verification.json",
              ROOT / "edge" / "runtime.py",
              ROOT / "edge" / "infer_on_device.py"]
    for path in needed:
        if not path.is_file():
            raise FileNotFoundError(f"먼저 이전 실습을 끝내세요: {path}")
    OUT.mkdir(parents=True, exist_ok=True)
    shutil.copy2(ARTIFACT / "tinycnn.onnx", OUT / "tinycnn.onnx")
    shutil.copy2(ARTIFACT / "training_config.json", OUT / "training_config.json")
    shutil.copy2(ROOT / "edge" / "runtime.py", OUT / "runtime.py")
    shutil.copy2(ROOT / "edge" / "infer_on_device.py", OUT / "infer_on_device.py")
    weight = (OUT / "tinycnn.onnx").read_bytes()
    sha = hashlib.sha256(weight).hexdigest()
    config = json.loads((OUT / "training_config.json").read_text(encoding="utf-8"))
    manifest = {
        "model_id": config["model_id"], "version": config["model_version"],
        "onnx_sha256": sha, "classes": config["classes"],
        "input_shape": [1, 3, config["image_size"], config["image_size"]],
        "input_name": "image", "output_name": "logits", "opset": 17,
        "preprocess": "PIL RGB / bilinear / float32 [0,1] / NCHW",
        "threshold_demo": config["confidence_threshold_demo"],
        "purpose": "교육용 빨간 카드 탐지; 실제 산업안전/불량 판정용 아님",
    }
    (OUT / "manifest.json").write_text(json.dumps(manifest, indent=2, ensure_ascii=False), encoding="utf-8")
    print("6일차 이관 파일 준비 완료:", OUT)
    for path in sorted(OUT.iterdir()):
        print(" -", path.name)
    print("ONNX SHA256:", sha)


if __name__ == "__main__":
    main()
```

실행합니다.

```bash
python -m scripts.14_package_day05
find deploy/day05_raspberrypi -maxdepth 1 -type f | sort
cat deploy/day05_raspberrypi/manifest.json
```

정상적인 최종 구조입니다.

```text
deploy/day05_raspberrypi/
├── tinycnn.onnx
├── training_config.json
├── runtime.py
├── infer_on_device.py
└── manifest.json
```

지금은 같은 PC의 CPU에서 배포 코드를 먼저 확인할 수 있습니다. Test 이미지 경로를 **배포 폴더 외부에서 입력**하면 됩니다.

```bash
python deploy/day05_raspberrypi/infer_on_device.py \
  --image data/day05/images/test/normal/normal_test_001.jpg
```

이 명령에서 사용하는 **`normal_test_001.jpg`는 직접 촬영 스크립트의 기본 파일명**입니다. 제공 이미지로 실습한다면 해당 이미지의 실제 파일명으로 바꿉니다.

응답에는 `input_ref`, `predicted_label`, `confidence`, `decision`, `inference_ms`, `model_id`, `model_version`가 출력됩니다. **Raspberry Pi에 복사하는 것은 6일차**입니다. Pi가 ARM64·64비트 OS와 호환되는 ONNX Runtime 패키지를 설치할 수 있는지는 장치 환경에서 확인해야 합니다.

---

## PART 8. 자동 테스트·오류 확인·프로젝트 정리

### 실습 22. 학습 구조와 데이터 검증 로직을 자동 테스트합니다

**이번 테스트는 무엇을 확인하나요?** 클래스 순서, CNN 출력 Shape, RGB 전처리, 이미지 폴더 로드, 데이터 수량 검사, Split 누수 검출을 자동으로 확인합니다. **ONNX 결과의 실제 일치 검증은 앞서 `13_verify_onnx.py`로 별도 실행한 결과**를 사용합니다.

**파일: `tests/test_day05.py`**

```python
"""5일차의 모델·전처리·데이터 품질을 확인하는 자동 테스트."""
from pathlib import Path

import numpy as np
import pytest
import torch
from PIL import Image

from edge.dataset import ImageFolderDataset, image_to_tensor, validate_dataset
from edge.model import CLASS_NAMES, TinyCNN


def test_class_order_is_fixed():
    assert CLASS_NAMES == ["normal", "warning_red"]


def test_tinycnn_output_shape():
    model = TinyCNN().eval()
    with torch.inference_mode():
        output = model(torch.zeros(1, 3, 64, 64))
    assert tuple(output.shape) == (1, 2)


def test_rgb_preprocessing_range_and_layout():
    picture = Image.new("RGB", (120, 90), (255, 0, 0))
    x = image_to_tensor(picture)
    assert tuple(x.shape) == (3, 64, 64)
    assert x.dtype == torch.float32
    assert 0 <= x.min().item() <= x.max().item() <= 1
    assert x[0].mean().item() > 0.99


def build_small_fixture(root: Path) -> None:
    for split in ("train", "val", "test"):
        for label in CLASS_NAMES:
            folder = root / split / label
            folder.mkdir(parents=True)
            # 분할 간 서로 다른 색상 → 파일 해시도 달라야 함
            value = (len(split) * 20 + (0 if label == "normal" else 30))
            Image.new("RGB", (25, 25), (value, 10, 50)).save(folder / "sample.png")


def test_small_image_folder_dataset(tmp_path):
    build_small_fixture(tmp_path)
    samples = ImageFolderDataset(tmp_path, "train")
    assert len(samples) == 2
    x, y = samples[0]
    assert tuple(x.shape) == (3, 64, 64)
    assert y == 0


def test_validate_dataset_counts(tmp_path):
    build_small_fixture(tmp_path)
    counts = validate_dataset(tmp_path, {"train": 1, "val": 1, "test": 1})
    assert counts["train"]["normal"] == 1
    assert counts["test"]["warning_red"] == 1


def test_reject_duplicate_image_across_splits(tmp_path):
    build_small_fixture(tmp_path)
    (tmp_path / "val" / "normal" / "sample.png").write_bytes(
        (tmp_path / "train" / "normal" / "sample.png").read_bytes()
    )
    with pytest.raises(ValueError, match="데이터 누수"):
        validate_dataset(tmp_path, {"train": 1, "val": 1, "test": 1})


def test_reject_missing_class_folder(tmp_path):
    with pytest.raises(FileNotFoundError):
        validate_dataset(tmp_path, {"train": 1, "val": 1, "test": 1})
```

프로젝트 루트에서 실행합니다.

```bash
python -m pytest -q tests/test_day05.py
```

예상 출력은 `7 passed`입니다. 이 테스트는 데이터·모델 코드의 기본 기능을 검사합니다. 이어서 **1~4일차 테스트를 포함해 기존 기능이 유지되는지도 확인**합니다.

```bash
python -m pytest -q
```

기존 일차 테스트에서는 PostgreSQL의 시험용 SQLite DB나 외부 API 대역을 사용한 경우도 있으므로 **전체 테스트 통과와 실제 카메라·PostgreSQL·Redis 연결 성공을 같은 의미로 해석하지 않습니다.** 장비 확인은 각 일차의 실제 실행 절차를 따릅니다.

### 실습 23. 오류가 발생하면 해당 단계부터 점검합니다

| 증상 | 확인할 내용 | 해결 방법 |
|---|---|---|
| `ModuleNotFoundError: torch / onnx / onnxruntime` | 5일차 패키지 설치 및 Python 실행 경로 | 같은 `.venv`에서 `python -m pip install -r requirements.txt` |
| `No module named edge` | 프로젝트 최상위에서 실행하지 않음 | `cd ~/ai_vision/subject13_ai_platform` 후 `python -m scripts.파일명` 사용 |
| 카메라 Open 실패 | IP Webcam 주소·Wi-Fi·연결 점유 | 앱의 현재 `/video` URL 확인, 4일차 Bridge 중지 후 재시도 |
| 촬영 시 기존 이미지 때문에 중지 | 자동 덮어쓰기 방지 | 이미지 백업 및 해당 폴더 정리 후 촬영 |
| 데이터 검사에서 누락/중복 | Split 분리 잘못됨 | 촬영 폴더/라벨 확인, 동일한 이미지는 제거하고 새로 촬영 |
| TinyCNN Accuracy 낮음 | 작은 데이터·촬영 조건·라벨 오류 | **Test를 보며 재학습하지 말고** Train/Val 품질을 확인하고 한계 기록 |
| ONNX Export 예외 | `onnx` 설치·PyTorch Export 호환성 | 의존성·출력 로그 확인, 모델·입력 규격 점검 |
| ONNX/PyTorch 결과 불일치 | RGB/BGR·입력 Shape·가중치 | `edge/dataset.py`와 `edge/runtime.py`의 전처리 비교 |
| 웹에서 HTTP 503 | `tinycnn.onnx` 또는 설정파일 없음 | 11~13번의 학습·ONNX 변환 순서 확인 |
| 웹에서 HTTP 422 | 이미지가 아니거나 전송 형태 오류 | JPG/PNG 파일 선택, 업로드 상한 5MB 확인 |
| Jetson Pose Export 오류 | 교과10 `.venv`의 Ultralytics·ONNX 의존성 | 원본 PT 존재·실행 Python·패키지 호환성 확인 |
| Raspberry Pi 실행 안 됨 | Pi OS/CPU 아키텍처·Wheel 호환성 | **6일차 실장비 환경에서** Python·ONNX Runtime 호환 버전 점검 |

모든 오류를 새 프로젝트 생성으로 해결하려 하지 말고 **입력 데이터 → Python 환경 → 학습 → ONNX 변환 → CPU 추론 → FastAPI·웹** 순서로 발생 위치를 확인합니다.

### 실습 24. 오늘의 설계 변경을 기록합니다

**파일: `docs/architecture.md` — 기존 내용 맨 아래에 추가**

```markdown
## 5일차 — Edge 모델 준비

- 1~4일차 FastAPI/PostgreSQL/Redis-Celery/Live OCR·Pose는 유지한다.
- 교육용 normal/warning_red 이미지는 data/day05/images/의 Train/Val/Test로 분리한다.
- PyTorch TinyCNN Best 체크포인트: artifacts/day05/tinycnn_best.pt
- Raspberry Pi ONNX 후보: artifacts/day05/tinycnn.onnx
- 입력 계약: RGB, 64×64, float32 0~1, NCHW [1,3,64,64].
- 출력 계약: logits [1,2], 클래스 순서 normal=0 / warning_red=1.
- FastAPI Preview: /api/v1/edge/health, /predict, /page. Preview는 DB에 저장하지 않는다.
- 6일차 반입: deploy/day05_raspberrypi/ ONNX·설정·추론 코드·Manifest.
- Jetson 후보: artifacts/day05/jetson/yolo11n-pose.onnx (교과 10 재사용).
- 9일차 TensorRT 엔진은 Jetson 장치에서 별도로 생성한다.
```

**파일: `README.md` — 기존 내용 맨 아래에 추가**

```markdown
## 5일차 — TinyCNN과 ONNX 배포 준비

1. 기존 프로젝트 `.venv`를 열고 requirements.txt에 5일차 패키지를 설치한다.
2. data/day05/images/{train,val,test}/{normal,warning_red}/에 이미지를 준비한다.
3. python -m scripts.10_check_day05_data
4. python -m scripts.11_train_tinycnn --epochs 12 --batch-size 8
5. python -m scripts.12_export_tinycnn_onnx
6. python -m scripts.13_verify_onnx
7. python -m scripts.14_package_day05
8. python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
9. 브라우저에서 http://127.0.0.1:8000/api/v1/edge/page 확인
10. 교과 10 `.venv`에서 scripts/15_prepare_jetson_pose.py로 YOLO Pose ONNX 후보 Export

개인 사진·원본 데이터·가중치·ONNX 파일·환경변수는 공개 Git에 업로드하지 않는다.
```

### 실습 25. `.gitignore`에 모델·이미지·배포 산출물을 추가합니다

이전 일차에 작성한 `.gitignore`를 지우지 말고 다음 항목을 확인해 **없는 줄만 추가**합니다.

**수정 파일: `.gitignore` — 맨 아래에 추가**

```gitignore
# 5일차 개인 촬영 이미지 / 모델 / 배포 파일
data/day05/images/
artifacts/day05/
reports/day05/
deploy/day05_raspberrypi/
```

프로젝트 원본 코드인 `edge/`, `scripts/09~15_*.py`, `app/edge_api.py`, `web/edge.html`, `tests/test_day05.py`, README·문서는 Git에 추가합니다. **가중치와 촬영 이미지, 모델 설정 결과, 배포 패키지는 Git에 포함하지 않습니다.** GitHub 공개 여부는 강의실·기업의 보안 규정을 따릅니다.

---

# 6. 오늘 작성한 내용의 최종 정리 및 GitHub 업로드

## 6.1. 오늘의 최종 결과

오늘은 4일차까지 만든 `subject13_ai_platform`을 유지하면서 **Edge 장치에 배포할 새 모델을 직접 학습하고 CPU 추론 코드까지 구성**했습니다.

```text
5일차 촬영 데이터 (Train / Validation / Test)
                     ↓
              TinyCNN CPU 학습
                     ↓
        tinycnn_best.pt (Validation 선택)
                     ↓
               tinycnn.onnx
                     ↓
           ONNX Runtime CPU 검증
                     ↓
        FastAPI /api/v1/edge/predict
                     ↓
             브라우저 결과 확인
                     ↓
       Raspberry Pi 이관 패키지 완성

[별도 교과10] yolo11n-pose.pt → yolo11n-pose.onnx
                     ↓
          8~9일차 Jetson에서 재사용
```

이번 TinyCNN은 **빨간 카드 표시를 구별하는 교육용 2클래스 분류기**입니다. 교과 9의 OCR 검사 모델이나 교과 10의 동작인식 모델을 대체하지 않습니다. 또한 5일차의 ONNX 모델은 기존 교과 13 DB 데이터나 3일차 Bronze/Silver/Gold 내보내기 결과를 삭제·수정하지 않습니다.

## 6.2. 최종 자가 체크리스트

- [ ] 4일차 프로젝트와 `.venv`를 그대로 사용했습니다.
- [ ] 기존 FastAPI·DB·Redis/Celery Router를 삭제하지 않았습니다.
- [ ] `normal / warning_red` 이미지가 Train/Val/Test로 각각 준비되어 있습니다.
- [ ] `python -m scripts.10_check_day05_data`가 READY를 출력합니다.
- [ ] TinyCNN 체크포인트와 `training_config.json`이 생성되었습니다.
- [ ] ONNX 변환이 완료되고 `onnx.checker`가 통과했습니다.
- [ ] Test의 PyTorch·ONNX 예측 일치 검증 보고서를 저장했습니다.
- [ ] `/api/v1/edge/health`, `/api/v1/edge/predict`, `/api/v1/edge/page`를 확인했습니다.
- [ ] `deploy/day05_raspberrypi/`에 ONNX·설정·추론 코드·Manifest가 있습니다.
- [ ] 교과 10의 YOLO Pose ONNX 후보와 Manifest를 확인했습니다.
- [ ] `python -m pytest -q tests/test_day05.py`의 7개 테스트가 통과했습니다.
- [ ] 개인 데이터·모델·비밀정보를 Git 업로드 대상에서 제외했습니다.
- [ ] README·구조도·5일차 Git 커밋을 완료했습니다.

## 6.3. Git에 오늘 작업을 기록합니다

프로젝트 루트에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git status --short

git add requirements.txt .gitignore README.md docs/architecture.md \
  app/main.py app/edge_api.py \
  edge/__init__.py edge/model.py edge/dataset.py edge/runtime.py edge/infer_on_device.py \
  scripts/09_capture_day05.py scripts/10_check_day05_data.py \
  scripts/11_train_tinycnn.py scripts/12_export_tinycnn_onnx.py \
  scripts/13_verify_onnx.py scripts/14_package_day05.py \
  scripts/15_prepare_jetson_pose.py \
  web/edge.html tests/test_day05.py

git status
```

`git status`에 **`data/day05/images/`, `artifacts/day05/`, `reports/day05/`, `deploy/day05_raspberrypi/`, `.env`** 등이 Staged로 표시되지 않는지 확인합니다. 민감한 파일이 보이면 커밋하지 말고 먼저 제외 설정과 추적 여부를 점검합니다.

```bash
git commit -m "교과13 5일차: TinyCNN 학습 및 ONNX Edge 모델 배포 준비"
git log -1 --oneline
git status --short
```

## 6.4. 허용된 원격 저장소에 올립니다

1일차에 원격 저장소를 연결한 상태라면 다시 `git init`하거나 원격을 추가하지 않습니다.

```bash
git remote -v
git push origin main
```

외부 GitHub가 제한된 폐쇄망 수업이라면 기관이 허용한 **내부 GitLab/Gitea** 또는 로컬 Git만 사용합니다. 모델 가중치나 개인 촬영 이미지는 Git에 올리지 않으므로, 다음 날 장치 실습에 필요한 배포 폴더는 **교육장에서 승인한 안전한 파일 전달 방식**으로 별도 보관합니다.

## 6.5. 6일차 수업과 어떻게 연결되나요?

**다음 6일차에도 동일한 프로젝트를 이어서 사용합니다.** 오늘 만든 `deploy/day05_raspberrypi/`의 `tinycnn.onnx`, `training_config.json`, `runtime.py`, `infer_on_device.py`, `manifest.json`을 Raspberry Pi로 전달한 뒤, Raspberry Pi에서 실제로 ONNX Runtime CPU가 동작하는지 확인합니다. 이후 카메라 영상 입력, `normal/ warning_red` 판정에 따른 파란색·빨간색 LED(GPIO), 결과 JSON의 PC FastAPI 전송과 오류 복구까지 단계적으로 확장합니다.

**이제 5일차 모델 학습 및 배포 준비가 끝났습니다. 다음 날에는 오늘 준비한 모델을 다시 학습하지 않고, Raspberry Pi에서 실제로 실행하도록 연결합니다.**
