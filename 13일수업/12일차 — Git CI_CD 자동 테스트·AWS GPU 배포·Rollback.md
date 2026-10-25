## 1. 오늘의 목표

11일차에는 기존 AI 플랫폼을 AWS EC2 GPU 서버에 직접 배포했습니다. FastAPI, PostgreSQL, Redis, Celery, Prometheus, Grafana와 YOLO11n-Pose CUDA 추론이 Docker Compose에서 함께 실행되도록 연결했습니다.

하지만 매번 코드를 변경할 때마다 EC2에 접속해 파일을 복사하고 명령어를 다시 실행하면, 실수로 잘못된 버전을 배포하거나 운영 중인 기능을 망가뜨릴 수 있습니다.

오늘은 **Git에 기록한 코드의 자동 검사(CI)와 AWS에 배포하는 과정(CD)을 연결**합니다. 새로운 AI 모델을 학습하지 않고, 11일차에 만든 실제 서비스를 그대로 사용합니다.

오늘 수업을 마치면 다음을 직접 수행할 수 있습니다.

- 11일차 Docker Compose·Cloud GPU 서비스를 삭제하지 않고 기존 프로젝트를 이어서 수정할 수 있습니다.
- 실제로 배포 중인 Git 커밋 SHA를 FastAPI API로 확인할 수 있습니다.
- GitHub Actions에서 Python 테스트·Compose 설정 검사·CPU Docker 이미지 빌드를 자동 실행할 수 있습니다.
- AWS OIDC 역할과 SSM을 이용해 **장기 AWS 키 또는 외부 공개 SSH 포트 없이** EC2 배포 작업을 요청할 수 있습니다.
- 배포 직전에 PostgreSQL 백업을 만들고, 배포 후 FastAPI·Cloud GPU·DB 조회의 정상 상태를 확인할 수 있습니다.
- 새 버전의 빌드 또는 상태 확인에 실패할 때 이전 Git 커밋으로 돌아가는 Rollback을 검증할 수 있습니다.
- 코드·배포 문서·Git 이력을 기록하고, 13일차 최종 인수검사로 이어갈 수 있습니다.

**오늘의 완성 기준:** `main`의 특정 40자리 Git SHA에 대해 CI가 통과하고, AWS SSM 배포가 성공했을 때 EC2의 `/api/v1/deploy/version`이 **바로 그 SHA**를 반환해야 합니다. `/api/v1/cloud/health`에서는 `connected=true`, `runtime=pytorch_cuda`를 확인합니다. PostgreSQL과 모델 가중치·비밀키는 변경·삭제하지 않습니다.

이전 교과에서 사용한 실제 모델과 이미지가 없는 경우에도 **AWS 배포 실행을 임의의 모의 추론으로 대신하지 않습니다.** CI와 Rollback 코드 검증은 진행할 수 있지만 실제 AWS 배포 확인은 필요한 자료가 준비된 뒤 수행합니다.

---

## 2. 오늘 수업의 전체 흐름

![[Pasted image 20261018133003.png]]

```text
[11일차의 실제 프로젝트]
FastAPI + PostgreSQL + Redis + Celery
Prometheus + Grafana + Cloud GPU YOLO11n-Pose
                         ↓
                 기존 파일·DB 확인
                         ↓
              Git 배포 버전 API 추가
                         ↓
               Compose 배포 설정 추가
                         ↓
             GitHub Actions Workflow
                         ↓
                 [CI 자동 검증]
        Python 회귀 테스트 · Compose 검사
               CPU Docker 이미지 빌드
                         ↓
                    CI 통과?
             ┌───────────┴───────────┐
            아니요                예
             ↓                     ↓
        배포 중단            AWS OIDC 인증
                                   ↓
                             SSM 배포 명령
                                   ↓
                         [기존 AWS EC2 서버]
                       원래 Git 커밋·DB 보존
                                   ↓
                         PostgreSQL 백업
                                   ↓
                         지정 커밋 Checkout
                                   ↓
                      API + GPU 이미지 Build
                                   ↓
                         Docker Compose 시작
                                   ↓
                    FastAPI·GPU·DB·SHA 점검
                             ┌─────┴─────┐
                            통과       실패
                             ↓           ↓
                        배포 확정   이전 커밋 Rollback
                             ↓           ↓
                        운영 기록     실패 원인 기록
                                   ↓
                     13일차 Acceptance Test
```

**중요:** 오늘은 7일차부터 사용한 네임드 볼륨과 DB 테이블을 그대로 유지합니다. Rollback은 **애플리케이션 코드·이미지를 되돌리는 작업**이며, PostgreSQL에 이미 기록된 업무 데이터를 자동으로 과거 상태로 되돌리는 작업이 아닙니다.

---

## 3. 실습 전에 알아야 할 핵심

### 3.1. 이전 수업에서 필요한 완성 결과

| 이어받는 일차 | 필요한 파일·기능 | 오늘 활용하는 이유 |
|---|---|---|
| 2일차 | `app/models.py`, `app/db.py`, `inference_events` 테이블 | 배포 전후 DB 데이터를 유지하고 조회 |
| 3일차 | Redis·Celery Worker, 기존 비동기 작업 | 배포 후 Worker가 계속 실행되는지 확인 |
| 7일차 | `Dockerfile`, `compose.yaml`, `scripts/11_dc.sh`와 공용 이미지 | 기존 API·Worker·DB 실행 구조 그대로 사용 |
| 10일차 | `compose.monitoring.yaml`, `monitoring/` | Prometheus·Grafana 유지 |
| 11일차 | `compose.cloud.yaml`, `compose.cloud.monitoring.yaml`, `cloud_gpu/`, `app/cloud_api.py` | EC2 GPU 추론·모니터링 유지 |
| 11일차 | EC2 `.env`, `artifacts/day05/jetson/yolo11n-pose.pt` | 기존 비밀키와 진짜 YOLO 모델 재사용 |

11일차 서비스를 완성하지 못했다면 **실제로 실행되던 11일차의 코드와 Compose 설정, 승인된 모델 파일, EC2의 `.env` 및 PostgreSQL 백업**을 먼저 복구해야 합니다. 모델이나 DB 내용을 새 예제 값으로 바꾸지 않습니다. 지난 수업의 전체 GitHub ZIP을 다시 다운로드하거나 다른 새 프로젝트를 만드는 실습은 아닙니다.

### 3.2. CI, CD, Rollback은 각각 무엇인가요?

- **CI(Continuous Integration):** 변경한 코드를 자동으로 검사합니다. 오늘은 `pytest`, Compose 설정 검사, API·Worker용 CPU Docker 이미지 빌드를 실행합니다.
- **CD(Continuous Delivery/Deployment):** CI를 통과한 **정확한 Git SHA**를 AWS EC2에 배포합니다. 오늘은 AWS Systems Manager의 Run Command가 EC2 내부 배포 스크립트를 실행합니다.
- **Rollback:** 새 버전의 배포가 실패하면 이전 Git SHA로 다시 빌드·시작합니다. **DB 데이터·Docker 볼륨을 삭제하지 않습니다.**

GitHub Actions의 CI 환경에는 NVIDIA GPU가 없을 수 있습니다. 따라서 **Cloud GPU CUDA 추론과 GPU 이미지 빌드는 실제 GPU가 있는 EC2에서만 진행**합니다. CI에서 Python 테스트가 모두 통과해도 실제 CUDA 실행 성공까지 확인된 것은 아닙니다.

### 3.3. 11일차의 EC2에는 Git 저장소가 없을 수 있습니다

11일차에는 EC2로 소스코드를 `rsync`하면서 `.git/`을 제외했습니다. 따라서 EC2에서 `git rev-parse HEAD`가 실패할 수 있습니다. 오늘은 EC2의 **기존 파일과 `.env`, 모델, DB 볼륨을 그대로 둔 상태로 Git 원격 연결을 처음 한 번 준비**합니다. 이전 파일을 지우는 `git clean -fdx`, 모델을 지우는 명령, `docker compose down -v`는 사용하지 않습니다.

### 3.4. 왜 SSH 비밀키를 GitHub Actions에 넣지 않나요?

AWS OIDC는 GitHub Actions에 AWS 장기 액세스 키를 저장하지 않고, GitHub에서 발급한 신원 토큰으로 일시적인 AWS 권한을 요청하는 방법입니다. SSM은 EC2에서 명령을 실행하므로 GitHub Actions의 실행 IP를 위해 SSH `22` 포트를 전체 인터넷에 열지 않아도 됩니다.

단, **EC2에 SSM Agent가 온라인이어야 하고, EC2용 IAM 역할 및 GitHub Actions용 IAM 역할을 사전에 준비해야 합니다.** GitHub 저장소에는 AWS 역할 ARN과 EC2 ID처럼 비밀이 아닌 설정값만 사용합니다. 실제 키와 비밀번호는 AWS EC2의 보호된 `.env`에 유지합니다.

### 3.5. 실제 프로젝트와 GitHub 저장소의 정책

이 실습은 **교육기관에서 GitHub와 AWS 사용을 허용한 경우**에 실행합니다. 폐쇄망에서 외부 GitHub가 금지되어 있다면 허용된 내부 GitLab/Gitea의 CI/CD와 배포 경로를 별도로 구성해야 합니다. GitHub에 몰래 코드를 올리거나 외부망 제한을 우회하지 않습니다. 이 교재의 실제 Workflow 파일은 **GitHub Actions용**이며 다른 CI 서비스에 그대로 붙여 넣는다고 작동하지 않습니다.

### 3.6. 실제 AWS 비용과 운영 데이터

EC2 GPU는 실행 시간이 길어지면 비용이 발생합니다. 수업 전 예산과 GPU 할당량을 확인하고, 사용이 끝난 후 EC2를 Stop/Terminate했는지 확인합니다. 배포 중에는 PostgreSQL 데이터와 모델을 삭제하지 않으며, 운영 데이터를 임의로 초기화하지 않습니다.

---

## 4. 오늘 사용할 기술 스택

| 기술 | 오늘의 실습 역할 |
|---|---|
| Git · Git SHA | 소스 버전 식별·Checkout·변경 이력 |
| GitHub Actions | `main` 푸시 시 Python 테스트와 Docker 이미지 빌드 |
| Python · Pytest · PyYAML | FastAPI·배포 명령·Compose 설정 자동 검증 |
| Docker · Docker Compose | 11일차 API·Worker·Cloud GPU 이미지 재빌드 및 배포 |
| AWS IAM · OIDC | GitHub Actions의 AWS 임시 인증 |
| AWS Systems Manager(SSM) | EC2에 원격 명령 전달·실행 상태 확인 |
| PostgreSQL `pg_dump` | 배포 전 업무 기록 백업 |
| FastAPI · HTTPX · JSON | 현재 배포 커밋과 API·GPU 상태 확인 |
| Bash · `flock` | 하나의 배포만 실행하고 실패 시 자동 복구 |
| Prometheus · Grafana | 배포 이후 기존 지표 유지 여부 확인 |

**준비 환경:** 기존 개발 PC(Windows 11 + WSL Ubuntu), 11일차 AWS EC2 GPU 인스턴스, 승인된 Git 저장소, VS Code, Python 3.12, Git, Docker Compose, AWS 콘솔 접속 권한입니다.

---

# 5. 처음부터 끝까지 따라 하는 상세 실습

## PART 1. 11일차 프로젝트를 확인하고 12일차 파일을 준비합니다

### 실습 1. 기존 프로젝트를 다시 엽니다

**이번 실습은 무엇을 하나요?** 지난 시간의 프로젝트를 그대로 열어 현재 어떤 기능까지 완성했는지 확인합니다. 새 프로젝트를 만들거나 기존 Docker 설정을 삭제하지 않습니다.

**기존 PC WSL Ubuntu 터미널**에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
pwd
source .venv/bin/activate
ls -l app/main.py app/cloud_api.py app/cloud_repo.py
ls -l compose.yaml compose.monitoring.yaml compose.cloud.yaml compose.cloud.monitoring.yaml
ls -l scripts/11_dc.sh
ls -lh artifacts/day05/jetson/yolo11n-pose.pt
git status
code .
```

`pwd`의 마지막이 `subject13_ai_platform`이고, 위 파일이 존재하는지 확인합니다. 11일차에 `scripts/11_dc.sh`를 실행했다면 이 파일에서 Compose 설정 **4개를 순서대로 결합**하는 것을 확인할 수 있습니다.

EC2가 현재 실행 중이고 SSH 접속이 가능하다면 **EC2 터미널**에서 다음도 확인합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
bash scripts/11_dc.sh ps
curl -fsS http://127.0.0.1:8000/health
curl -fsS http://127.0.0.1:8000/api/v1/cloud/health
```

`/api/v1/cloud/health`가 `connected: true`를 반환한다면 Cloud GPU와 API 연결이 정상입니다. **EC2가 중지된 상태라면** 먼저 AWS 콘솔에서 승인을 받고 시작한 다음 확인합니다. EC2가 없거나 11일차 기능이 정상 실행되지 않으면 배포 전 단계부터 확인합니다.

### 실습 2. 실제 DB·모델·비밀키가 보존되어 있는지 확인합니다

**EC2 터미널**에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
ls -lh artifacts/day05/jetson/yolo11n-pose.pt
ls -l .env
bash scripts/11_dc.sh exec -T db   sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT COUNT(*) FROM inference_events;"'
```

출력된 DB 기록 건수를 따로 메모합니다. 데이터 건수는 학생마다 다르며 특정 숫자를 맞출 필요는 없습니다. **배포 후에도 기존 기록이 유지되었는지 비교**하기 위한 값입니다.

**주의:** `.env`의 실제 비밀번호를 화면에 출력하거나 캡처해 공유하지 않습니다. 위 명령은 파일 존재와 권한만 확인합니다.

### 실습 3. 11일차의 마지막 정상 Git 커밋을 확인합니다

**기존 PC 터미널**에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git branch --show-current
git status
git log -3 --oneline
export DAY11_SHA="$(git rev-parse HEAD)"
printf '11일차 마지막 커밋: %s
' "$DAY11_SHA"
```

이 SHA는 EC2가 기존에 실행하던 **11일차 소스 버전**을 식별하는 데 사용합니다. 오늘 새 코드를 커밋하기 전에 확인해야 기존 서버 코드와 정확하게 대응할 수 있습니다. SHA는 임의로 만들어 입력하지 않습니다.

### 실습 4. 12일차 파일·폴더를 만듭니다

**기존 PC 프로젝트 루트**에서 실행합니다.

```bash
mkdir -p .github/workflows deploy/day12 scripts tests docs
```

오늘 완성할 신규 파일은 아래와 같습니다. 기존 `app/main.py`는 **파일 전체를 교체하지 않고 Router 연결 두 줄만 추가**합니다.

```text
subject13_ai_platform/
├── .github/workflows/
│   └── day12-ci-cd.yml                # CI/CD Workflow [새 파일]
├── app/
│   ├── main.py                        # 기존 FastAPI [2줄 추가]
│   └── version_api.py                 # 배포 커밋 API [새 파일]
├── compose.deploy.yaml               # 기존 Compose에 배포 SHA 추가 [새 파일]
├── scripts/
│   ├── 11_dc.sh                       # 11일차 그대로 유지
│   ├── 12_dc.sh                       # 12일차 Compose 실행기 [새 파일]
│   ├── 12_release_check.py            # 실제 배포 상태 확인 [새 파일]
│   ├── 12_ci_verify.py                # CI에서 추적 파일·필수 경로 점검 [새 파일]
│   └── 12_ssm_deploy.py               # AWS SSM 배포 호출 [새 파일]
├── deploy/day12/
│   └── remote_deploy.sh               # EC2에서 설치할 배포 실행기 [새 파일]
├── tests/
│   ├── test_day12_pipeline.py         # API·Workflow·SSM 계약 검증 [새 파일]
│   └── test_day12_remote_rollback.py  # 임시 Git으로 성공·실패 Rollback 검증
└── docs/
    └── day12_cicd_runbook.md          # 운영 기록·복구 절차 [새 파일]
```

기존 `compose.yaml`과 `compose.monitoring.yaml`, `compose.cloud.yaml`, `compose.cloud.monitoring.yaml`, DB·모델 파일은 그대로 보관합니다. 같은 폴더의 1~11일차 코드와 README도 삭제하지 않습니다.

---

## PART 2. 배포 버전 확인 API와 Docker Compose 설정을 추가합니다

### 실습 5. 지금 실행 중인 Git 커밋을 보여 주는 API를 작성합니다

**이번 코드는 무엇을 하나요?** PC의 Git에서 어떤 버전을 올렸는지와 EC2 컨테이너가 실제 어떤 버전으로 실행 중인지 비교할 수 있도록 합니다.

**의사코드**

```text
Docker Compose에서 APP_COMMIT_SHA 환경변수 전달
                  ↓
FastAPI에서 APP_COMMIT_SHA 읽기
                  ↓
40자리 Git SHA인지 확인
                  ↓
GET /api/v1/deploy/version
                  ↓
브라우저·자동 배포 프로그램에서 버전 확인
```

**파일: `app/version_api.py`**

```python
"""12일차: 실제 배포된 Git 커밋 정보를 확인하는 API입니다."""
import os
import re
from fastapi import APIRouter

router = APIRouter(tags=["day12-release"])
_COMMIT = re.compile(r"^[0-9a-f]{40}$")


def current_release() -> dict[str, str]:
    sha = os.getenv("APP_COMMIT_SHA", "").lower()
    if _COMMIT.fullmatch(sha):
        return {"commit": sha, "source": "git", "service": "subject13-platform"}
    return {"commit": "unknown", "source": "local", "service": "subject13-platform"}


@router.get("/api/v1/deploy/version")
def get_deploy_version():
    return current_release()
```

`APP_COMMIT_SHA`는 학습 모델의 버전이 아니라 **서비스 소스코드의 Git 커밋 ID**입니다. 아직 Docker에서 전달하지 않은 개발 PC에서는 `commit: unknown`, `source: local`로 응답할 수 있습니다. 이 값은 자동 배포의 성공 판정에 사용됩니다.

### 실습 6. 기존 FastAPI에 새 Router를 연결합니다

**파일: `app/main.py` — 수정할 부분만 추가합니다.** 1~11일차에 작성한 다른 Router와 `FastAPI` 생성 코드를 그대로 유지합니다.

기존 Import 영역에 다음 한 줄을 추가합니다.

```python
from app.version_api import router as version_router
```

그리고 **기존 `app = FastAPI(...)`를 만든 뒤 다른 `include_router(...)`를 연결하던 위치**에 다음 한 줄을 추가합니다.

```python
app.include_router(version_router)
```

`app = FastAPI(...)`를 새로 하나 더 만들면 이전 API가 끊어질 수 있으므로 **기존 `app` 객체를 사용**합니다. 파일을 저장한 뒤 다음 명령으로 새 Router의 독립 실행을 확인합니다.

```bash
python - <<'PY'
from fastapi import FastAPI
from fastapi.testclient import TestClient
from app.version_api import router
api = FastAPI()
api.include_router(router)
result = TestClient(api).get('/api/v1/deploy/version')
print(result.status_code, result.json())
PY
```

HTTP 200과 `commit`, `source`, `service` 항목이 나타나면 Router 자체는 정상입니다. 오늘의 배포 이전에는 `commit: unknown`일 수 있습니다.

### 실습 7. 기존 Docker Compose에 배포 SHA를 전달합니다

**이번 실습은 무엇을 하나요?** 기존 7·10·11일차 Compose 파일은 수정하지 않고, 12일차의 새 Overlay를 덧붙입니다. PostgreSQL·Redis·Grafana의 기존 네임드 볼륨은 바꾸지 않습니다.

**의사코드**

```text
기존 Compose 4개 유지
          ↓
compose.deploy.yaml 추가
          ↓
FastAPI와 Cloud GPU 컨테이너에 APP_COMMIT_SHA 전달
          ↓
기존 데이터베이스·모델·모니터링 유지
```

**파일: `compose.deploy.yaml`**

```yaml
# 12일차 추가 Compose: 기존 11일차 서비스와 볼륨 이름을 변경하지 않습니다.
services:
  api:
    environment:
      APP_COMMIT_SHA: ${APP_COMMIT_SHA:?scripts/12_dc.sh 로 실행하세요}
  cloud-gpu:
    environment:
      APP_COMMIT_SHA: ${APP_COMMIT_SHA:?scripts/12_dc.sh 로 실행하세요}
```

`compose.deploy.yaml`은 새로운 데이터베이스를 생성하지 않고, API와 Cloud GPU에 환경변수만 추가합니다.

### 실습 8. Docker Compose 실행 스크립트를 작성합니다

앞서 11일차에는 `scripts/11_dc.sh`로 네 개의 Compose 설정을 병합했습니다. 이제 12일차 Overlay를 마지막에 추가합니다.

**의사코드**

```text
현재 프로젝트 경로 확인
        ↓
git rev-parse HEAD로 현재 SHA 읽기
        ↓
APP_COMMIT_SHA 환경변수 설정
        ↓
기존 네 개 + 신규 Overlay 하나 순서로 결합
        ↓
입력한 Docker Compose 명령 실행
```

**파일: `scripts/12_dc.sh`**

```bash
#!/usr/bin/env bash
# 11일차 4개 Compose 구성을 보존하면서 12일차 배포 버전 정보만 추가합니다.
set -euo pipefail
ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$ROOT"
export APP_COMMIT_SHA="$(git rev-parse HEAD)"
exec docker compose \
  -f compose.yaml \
  -f compose.monitoring.yaml \
  -f compose.cloud.yaml \
  -f compose.cloud.monitoring.yaml \
  -f compose.deploy.yaml \
  "$@"
```

스크립트 저장 후 실행 권한을 부여하고 Bash 구문을 검사합니다.

```bash
chmod +x scripts/12_dc.sh
bash -n scripts/12_dc.sh
```

**기존 PC WSL 터미널**에서 아래 명령을 실행합니다.

```bash
bash scripts/12_dc.sh config --quiet
bash scripts/12_dc.sh config --services
```

`db`, `redis`, `api`, `worker`, `prometheus`, `grafana`, `cloud-gpu`가 출력되면 기존 서비스가 유지된 것입니다. `.env` 오류가 나타나면 11일차에서 사용한 값을 확인합니다. **새로 만든 예시 비밀번호를 실제 운영 DB 비밀번호 대신 넣지 않습니다.**

---

## PART 3. 실제 배포 확인·AWS 배포 요청 코드를 작성합니다

### 실습 9. 배포된 API·GPU·DB 상태를 검사합니다

**이번 코드는 무엇을 하나요?** Docker 프로세스가 단순히 켜져 있는 것뿐 아니라, 실제 Git SHA와 Cloud GPU 연결 및 PostgreSQL 조회가 일치하는지 확인합니다.

**의사코드**

```text
배포할 Git SHA 받기
          ↓
GET /health 로 FastAPI 정상 확인
          ↓
GET /api/v1/deploy/version 과 SHA 비교
          ↓
GET /api/v1/cloud/health 의 connected=true·CUDA 확인
          ↓
GET /api/v1/cloud/events?limit=1 의 DB 목록 확인
          ↓
모두 성공 → 배포 확정
하나라도 실패 → 재확인 후 오류 반환
```

**파일: `scripts/12_release_check.py`**

```python
"""12일차: 실제 배포 커밋, DB 조회 API, Cloud GPU 연결을 검증합니다.

모델 추론 자체는 /cloud 에서 실제 이미지를 별도로 넣어 확인합니다.
"""
import argparse
import json
import re
import time
from urllib.request import urlopen

SHA_PATTERN = re.compile(r"^[a-f0-9]{40}$")


def read_json(url: str) -> dict:
    with urlopen(url, timeout=5) as response:
        if response.status != 200:
            raise RuntimeError(f"HTTP {response.status}: {url}")
        return json.load(response)


def verify_once(base_url: str, expected_sha: str) -> None:
    base = base_url.rstrip("/")
    health = read_json(base + "/health")
    if health.get("status") != "ok":
        raise AssertionError("FastAPI /health 상태가 ok가 아닙니다.")

    version = read_json(base + "/api/v1/deploy/version")
    if version.get("commit") != expected_sha:
        raise AssertionError("배포된 커밋이 요청한 커밋과 다릅니다.")

    cloud = read_json(base + "/api/v1/cloud/health")
    if cloud.get("connected") is not True or cloud.get("runtime") != "pytorch_cuda":
        raise AssertionError("Cloud GPU가 실제 CUDA 모드로 연결되지 않았습니다.")

    events = read_json(base + "/api/v1/cloud/events?limit=1")
    if not isinstance(events, list):
        raise AssertionError("Cloud 기록 조회가 JSON 목록을 반환하지 않았습니다.")


def check_release(base_url: str, expected_sha: str, attempts: int, interval: float) -> None:
    if not SHA_PATTERN.fullmatch(expected_sha):
        raise ValueError("Git SHA는 40자리 소문자 16진수여야 합니다.")
    if attempts < 1 or interval < 0:
        raise ValueError("attempts는 1 이상, interval은 0 이상이어야 합니다.")
    error = None
    for index in range(attempts):
        try:
            verify_once(base_url, expected_sha)
            print("PASS: FastAPI + 배포 SHA + Cloud GPU + DB 조회")
            print("배포된 커밋:", expected_sha)
            return
        except (OSError, ValueError, AssertionError, RuntimeError) as exc:
            error = exc
            print(f"재확인 {index + 1}/{attempts}: {exc}")
            if index + 1 < attempts:
                time.sleep(interval)
    raise RuntimeError(f"배포 확인 실패: {error}")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--base-url", default="http://127.0.0.1:8000")
    parser.add_argument("--expected-sha", required=True)
    parser.add_argument("--attempts", type=int, default=18)
    parser.add_argument("--interval", type=float, default=10)
    args = parser.parse_args()
    check_release(args.base_url, args.expected_sha, args.attempts, args.interval)


if __name__ == "__main__":
    main()
```

코드에서 `urlopen()`으로 FastAPI에 접근합니다. **현재 배포 SHA와 요청한 SHA가 정확히 같아야 성공**합니다. `runtime=pytorch_cuda`도 확인하므로 CPU로만 실행된 Cloud 모델을 CUDA 성공으로 표시하지 않습니다.

이 코드는 12일차 배포를 마친 EC2에서 정상 동작을 최종 확인할 때 실행합니다. 작성한 직후에는 Docker에 새 Router가 아직 배포되지 않아 실패할 수 있습니다.

### 실습 10. CI에서 소스 파일과 보안 제외 항목을 검사합니다

비밀번호, 개인키, 모델 가중치와 DB 백업이 Git에 포함된 상태로 자동 배포되면 안 됩니다. **Git이 실제 추적하고 있는 파일**을 검사합니다.

**의사코드**

```text
12일차 필수 파일 존재 확인
          ↓
Python 소스 문법 검사
          ↓
git ls-files 로 추적 파일 확인
          ↓
.env / *.pem / 모델 / 백업 포함 여부 검사
          ↓
문제 발견 시 CI 실패
```

**파일: `scripts/12_ci_verify.py`**

```python
"""12일차: GitHub Actions가 필요한 소스·설정만 포함했는지 사전 검사합니다.

이 검사는 실제 AWS 배포 성공을 보증하지 않습니다.
"""
import py_compile
import subprocess
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]
REQUIRED = [
    "compose.yaml", "compose.monitoring.yaml", "compose.cloud.yaml",
    "compose.cloud.monitoring.yaml", "compose.deploy.yaml",
    "Dockerfile", "cloud_gpu/Dockerfile.cloud-gpu",
    "scripts/11_dc.sh", "scripts/12_dc.sh",
    "scripts/12_release_check.py", "scripts/12_ssm_deploy.py",
    "deploy/day12/remote_deploy.sh", ".github/workflows/day12-ci-cd.yml",
    "app/version_api.py", "app/cloud_api.py", "web/cloud.html",
]
BANNED_TRACKED = [".env", "backups/", "artifacts/", "handoff/records/", "handoff/provided/"]


def verify_layout(root: Path = ROOT) -> None:
    missing = [name for name in REQUIRED if not (root / name).is_file()]
    if missing:
        raise ValueError("필수 파일 누락: " + ", ".join(missing))


def verify_git_tracking(root: Path = ROOT) -> None:
    proc = subprocess.run(
        ["git", "ls-files", "-z"], cwd=root, capture_output=True, check=True
    )
    tracked = [raw.decode("utf-8", "replace") for raw in proc.stdout.split(b"\0") if raw]
    bad = [
        file for file in tracked
        if file in {".env", ".env.production"}
        or file.endswith((".pem", ".key", ".dump", ".pt", ".onnx", ".engine"))
        or file.startswith(tuple(prefix for prefix in BANNED_TRACKED if prefix.endswith("/")))
    ]
    if bad:
        raise ValueError("Git 추적 금지 파일 발견: " + ", ".join(bad[:20]))


def main() -> None:
    verify_layout()
    for file in (ROOT / "app/version_api.py", ROOT / "scripts/12_release_check.py",
                 ROOT / "scripts/12_ssm_deploy.py"):
        py_compile.compile(str(file), doraise=True)
    verify_git_tracking()
    print("PASS: 배포 필수 파일, Python 문법, Git 추적 금지 파일 확인")


if __name__ == "__main__":
    main()
```

`.gitignore`에 항목을 넣었다는 이유만으로 **이미 Git에 추적된 파일이 자동으로 제거되는 것은 아닙니다.** 이 검사에서 민감한 파일이 발견되면 배포를 중단하고 안전한 방법으로 기록을 정리합니다.

### 실습 11. AWS SSM 배포 요청 스크립트를 작성합니다

GitHub Actions에서 AWS OIDC 인증을 받고, AWS Systems Manager(SSM)로 EC2 내부의 배포 실행기를 호출합니다. **개인 SSH 키를 GitHub Actions에 저장하지 않습니다.**

**의사코드**

```text
GitHub가 검사한 40자리 SHA와 EC2 ID 확인
           ↓
AWS SSM send-command 실행
           ↓
EC2의 ~/subject13-deploy/remote_deploy.sh 호출
           ↓
AWS SSM get-command-invocation 반복 조회
           ↓
Success와 종료 코드 0 → 성공
그 외 → 배포 실패로 GitHub Actions에 반환
```

**파일: `scripts/12_ssm_deploy.py`**

```python
"""12일차 GitHub Actions: OIDC AWS 자격으로 SSM 배포 명령 전송·완료 확인.

실제 SSH 개인키, 장기 AWS 액세스 키를 Actions에 저장하지 않습니다.
"""
import argparse
import json
import re
import subprocess
import tempfile
import time
from pathlib import Path

SHA_PATTERN = re.compile(r"^[0-9a-f]{40}$")
INSTANCE_PATTERN = re.compile(r"^i-[0-9a-f]{8,17}$")
REMOTE_RUNNER = "/home/ubuntu/subject13-deploy/remote_deploy.sh"
FINISHED = {"Success", "Failed", "Cancelled", "TimedOut", "Cancelling"}


def run_aws(*arguments: str) -> dict:
    result = subprocess.run(
        ["aws", "--no-cli-pager", *arguments, "--output", "json"],
        check=True, capture_output=True, text=True
    )
    return json.loads(result.stdout)


def deploy_via_ssm(instance_id: str, commit_sha: str, timeout: int = 4800,
                   interval: int = 15) -> dict:
    if not INSTANCE_PATTERN.fullmatch(instance_id):
        raise ValueError("EC2 Instance ID가 올바르지 않습니다.")
    if not SHA_PATTERN.fullmatch(commit_sha):
        raise ValueError("40자리 Git 커밋 SHA가 필요합니다.")
    if timeout < 120 or interval < 1:
        raise ValueError("Timeout/Interval 설정을 확인하세요.")
    command = f"sudo -u ubuntu -H /bin/bash {REMOTE_RUNNER} {commit_sha}"
    params = {"commands": [command], "executionTimeout": [str(timeout)]}
    # params 파일에 AWS 키나 모델 데이터는 포함되지 않습니다.
    with tempfile.TemporaryDirectory(prefix="day12-ssm-") as tempdir:
        param_file = Path(tempdir) / "parameters.json"
        param_file.write_text(json.dumps(params), encoding="utf-8")
        result = run_aws(
            "ssm", "send-command", "--instance-ids", instance_id,
            "--document-name", "AWS-RunShellScript",
            "--comment", f"subject13 release {commit_sha[:12]}",
            "--parameters", f"file://{param_file}",
            "--timeout-seconds", "600", "--max-concurrency", "1", "--max-errors", "0",
        )
    command_id = result["Command"]["CommandId"]
    print(f"SSM 배포 명령 ID: {command_id}")
    deadline = time.monotonic() + timeout + 120
    while time.monotonic() < deadline:
        try:
            state = run_aws(
                "ssm", "get-command-invocation", "--command-id", command_id,
                "--instance-id", instance_id
            )
        except subprocess.CalledProcessError:
            # SSM API의 최종 일관성으로 명령 직후 조회가 잠시 실패할 수 있습니다.
            time.sleep(interval)
            continue
        status = state.get("Status", "Unknown")
        print("SSM 배포 상태:", status)
        if status in FINISHED:
            result_code = state.get("ResponseCode")
            print("원격 종료 코드:", result_code)
            print("원격 로그(일부):\n", state.get("StandardOutputContent", "")[-5000:])
            if status != "Success" or result_code != 0:
                print("원격 오류(일부):\n", state.get("StandardErrorContent", "")[-4000:])
                raise RuntimeError(f"배포 실패: {status} (exit={result_code})")
            return state
        time.sleep(interval)
    raise TimeoutError("SSM 배포가 제한 시간 안에 끝나지 않았습니다. 서버에서 상태를 확인하세요.")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--instance-id", required=True)
    parser.add_argument("--sha", required=True)
    parser.add_argument("--timeout", type=int, default=4800)
    args = parser.parse_args()
    deploy_via_ssm(args.instance_id, args.sha, timeout=args.timeout)


if __name__ == "__main__":
    main()
```

AWS `send-command`는 **명령 접수**와 **명령의 정상 종료**가 다릅니다. 따라서 SSM의 최종 실행 상태와 종료 코드를 함께 확인합니다. 실제 EC2에서 GPU 이미지를 빌드하는 데 시간이 걸릴 수 있어 반복 조회 간격을 둡니다.

### 실습 12. EC2에서 새 코드를 배포하고 실패하면 이전 버전으로 복구하도록 작성합니다

이 파일은 **실행 중인 Git 프로젝트 밖에 설치**할 배포 프로그램입니다. 새로 배포할 파일이 고장 나더라도 배포 실행기 자체가 지워지지 않도록 별도로 보존합니다.

**의사코드**

```text
Git SHA 유효성 확인
       ↓
flock으로 중복 배포 차단
       ↓
기존 .env·모델 보존 상태 확인
       ↓
main 브랜치에 있는 커밋인지 검증
       ↓
운영 PostgreSQL pg_dump 백업
       ↓
새 커밋 Checkout
       ↓
Compose 설정 검사·CPU/GPU 이미지 Build
       ↓
새 Compose 서비스 시작
       ↓
API·실제 GPU·DB·Git SHA 재확인
       ├─ 통과 → last_good_sha 저장
       └─ 실패 → 이전 SHA 재Checkout·재빌드·재기동
                             ↓
                    ROLLBACK_OK / FAILED
```

**파일: `deploy/day12/remote_deploy.sh`**

```bash
#!/usr/bin/env bash
# EC2에 한 번 설치해 계속 사용하는 배포 실행기: Git으로 배포될 코드와 분리합니다.
# SSM에서는 승인된 ubuntu 계정으로만 실행합니다.
set -Eeuo pipefail
umask 077

TARGET="${1:-}"
if [[ ! "$TARGET" =~ ^[0-9a-f]{40}$ ]]; then
  echo 'ERROR: 40자리 Git 커밋 SHA가 필요합니다.' >&2
  exit 2
fi
if [[ "$(id -un)" != "ubuntu" && -z "${SUBJECT13_ALLOW_TEST_USER:-}" ]]; then
  echo 'ERROR: 운영 배포는 ubuntu 계정에서만 실행합니다.' >&2
  exit 2
fi

ROOT="${SUBJECT13_APP_ROOT:-$HOME/ai_vision/subject13_ai_platform}"
HEALTH_BASE="${SUBJECT13_HEALTH_BASE:-http://127.0.0.1:8000}"
STATE="${SUBJECT13_DEPLOY_STATE:-$HOME/subject13-deploy/state}"
mkdir -p "$STATE/backups"
exec 9>"$STATE/deploy.lock"
if ! flock -n 9; then
  echo 'ERROR: 다른 배포 작업이 실행 중입니다.' >&2
  exit 3
fi

cd "$ROOT"
PREVIOUS="$(git rev-parse HEAD)"
STAGE="preflight"

check_old_health() {
  SUBJECT13_HEALTH_BASE="$HEALTH_BASE" python3 - <<'PY'
import json
import os
from urllib.request import urlopen
base = os.environ["SUBJECT13_HEALTH_BASE"].rstrip("/")
with urlopen(base + '/health', timeout=8) as r:
    if r.status != 200 or json.load(r).get('status') != 'ok':
        raise SystemExit('기존 API 상태 확인 실패')
with urlopen(base + '/api/v1/cloud/health', timeout=8) as r:
    data = json.load(r)
    if not data.get('connected'):
        raise SystemExit('기존 Cloud GPU 연결 확인 실패')
print('PASS: 기존 API + GPU 서비스 복구 확인')
PY
}

rollback() {
  local failed_code="$?"
  trap - ERR
  set +e
  echo "배포 실패(단계=$STAGE). 이전 커밋으로 복구합니다: ${PREVIOUS:0:12}" >&2
  git checkout --detach --force "$PREVIOUS"
  local checkout_status="$?"
  if (( checkout_status == 0 )); then
    # 11일차에는 12_dc.sh가 없으므로 11_dc.sh로 복원합니다.
    if [[ -f scripts/12_dc.sh ]]; then
      bash scripts/12_dc.sh build api cloud-gpu
      bash scripts/12_dc.sh up -d --no-build
    else
      bash scripts/11_dc.sh build api cloud-gpu
      bash scripts/11_dc.sh up -d --no-build
    fi
    local compose_status="$?"
    if (( compose_status == 0 )); then
      check_old_health
      local health_status="$?"
      if (( health_status == 0 )); then
        echo "ROLLBACK_OK: ${PREVIOUS}" >&2
      else
        echo 'ROLLBACK_FAILED: API/GPU 수동 복구가 필요합니다.' >&2
      fi
    else
      echo 'ROLLBACK_FAILED: Compose 복구 실패, 즉시 점검하세요.' >&2
    fi
  else
    echo 'ROLLBACK_FAILED: Git 이전 버전 복귀 실패, 즉시 점검하세요.' >&2
  fi
  # 복구에 성공해도 배포 실패 사실은 CI에 알립니다.
  exit "$failed_code"
}

# 실패한 중간 단계에서는 기존 서비스에 변화를 주기 전이므로 롤백하지 않습니다.
[[ -r .env ]] || { echo 'ERROR: EC2의 .env가 없습니다.' >&2; exit 2; }
[[ -s artifacts/day05/jetson/yolo11n-pose.pt ]] || {
  echo 'ERROR: 기존 YOLO Pose 모델 파일이 없습니다.' >&2; exit 2;
}
if ! git diff --quiet || ! git diff --cached --quiet; then
  echo 'ERROR: EC2 Git 파일에 미커밋 수정이 있습니다. 수동 확인 후 배포하세요.' >&2
  exit 2
fi

STAGE="fetch"
git fetch --no-tags origin main
git cat-file -e "${TARGET}^{commit}"
git merge-base --is-ancestor "$TARGET" refs/remotes/origin/main || {
  echo 'ERROR: main에 없는 Git 커밋은 배포하지 않습니다.' >&2
  exit 2
}

STAGE="backup"
STAMP="$(date -u '+%Y%m%dT%H%M%SZ')"
BACKUP="$STATE/backups/before_${TARGET:0:12}_${STAMP}.dump"
# PostgreSQL은 기존 11일차 Compose와 동일한 볼륨을 사용합니다.
bash scripts/11_dc.sh exec -T db sh -c \
  'exec pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc' > "$BACKUP"
[[ -s "$BACKUP" ]] && [[ "$(head -c 5 "$BACKUP")" == PGDMP ]] || {
  echo 'ERROR: DB 백업 파일 검증 실패' >&2
  exit 2
}
echo "기존 DB 백업 완료: $BACKUP"

# 이 시점부터 실패하면 서비스와 Git 코드를 이전 커밋으로 되돌립니다.
trap rollback ERR
STAGE="checkout"
git checkout --detach --force "$TARGET"

STAGE="configuration"
python3 scripts/11_check_cloud_inputs.py
bash scripts/12_dc.sh config --quiet

STAGE="build"
bash scripts/12_dc.sh build api cloud-gpu

STAGE="start"
bash scripts/12_dc.sh up -d --no-build

STAGE="verification"
python3 scripts/12_release_check.py \
  --base-url "$HEALTH_BASE" --expected-sha "$TARGET" --attempts 18 --interval 10

# 기존/신규 Docker 네임드 볼륨은 재생성하지 않습니다.
printf '%s\n' "$TARGET" > "$STATE/last_good_sha.tmp"
mv -f "$STATE/last_good_sha.tmp" "$STATE/last_good_sha"
trap - ERR
echo "DEPLOY_OK: ${TARGET}"
```

**중요한 코드 이해**

- `flock`은 동시에 여러 배포가 시작되지 않도록 제한합니다.
- `git merge-base --is-ancestor`는 확인한 `main`에 포함된 SHA만 배포하도록 제한합니다.
- `pg_dump -Fc`는 신규 이미지 배포 **직전에 기존 PostgreSQL 기록을 백업**합니다.
- `docker compose down -v`나 `docker volume rm` 명령은 없습니다. 기존 DB·Redis·Grafana 데이터는 자동으로 삭제하지 않습니다.
- `git checkout --detach --force`는 Git 추적 소스의 버전을 바꾸는 작업입니다. EC2의 `.env`·모델·백업은 Git에서 제외하고 보존합니다. 단, 추적 중인 파일에 미커밋 수정이 있으면 배포를 중단합니다.
- 새 코드 빌드 또는 상태 검증 실패 시 이전 SHA로 복귀합니다. 복귀 성공 후에도 **신규 배포 자체는 실패로 기록**합니다.

**이 실습의 Rollback 범위는 코드·이미지에 한정됩니다.** 운영 DB의 컬럼·테이블 변경을 자동으로 되돌리는 기능은 아직 없습니다. 12일차에는 기존 2일차부터 만든 PostgreSQL 스키마를 변경하지 않습니다.

```bash
chmod +x deploy/day12/remote_deploy.sh
bash -n deploy/day12/remote_deploy.sh
```

---

## PART 4. GitHub Actions와 자동 테스트를 작성합니다

### 실습 13. CI와 CD Workflow를 작성합니다

GitHub에 소스코드가 올라오면 **테스트 → Docker CPU 이미지 빌드 → AWS 인증 → EC2 실제 배포** 순서로 진행합니다. 초기 설정을 마치기 전에는 **CI만** 실행하도록 활성화 변수를 둡니다.

**의사코드**

```text
Git push main / Actions Run workflow
           ↓
CI: Python 회귀 테스트
           ↓
CI: Compose YAML 검사
           ↓
CI: Docker API/Worker CPU 이미지 Build
           ↓
실패 → 배포 중단
성공 → DAY12_DEPLOY_ENABLED=true 확인
           ↓
AWS OIDC 임시 권한 확보
           ↓
AWS SSM으로 EC2 실행기 호출
           ↓
실제 배포·Rollback 결과 GitHub에 기록
```

**파일: `.github/workflows/day12-ci-cd.yml`**

```yaml
name: Subject13 Day12 CI CD

'on':
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: subject13-cloud-production
  cancel-in-progress: false

jobs:
  ci:
    name: Python tests and CPU Docker build
    runs-on: ubuntu-latest
    env:
      DATABASE_URL: 'sqlite+pysqlite:///:memory:'
      DB_PASSWORD: ci-not-real-password
      DAY06_DEVICE_API_KEY: ci-not-real-device-key
      CLOUD_GPU_API_KEY: ci-not-real-gpu-key-long-enough
      GRAFANA_ADMIN_USER: ci
      GRAFANA_ADMIN_PASSWORD: ci-not-real-grafana-password
    steps:
      - name: Fetch current commit
        uses: actions/checkout@v4
      - name: Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip
          cache-dependency-path: requirements.docker.txt
      - name: Install test dependencies
        run: |
          python -m pip install --upgrade pip
          python -m pip install -r requirements.docker.txt pyyaml
      - name: Regression tests for days 8 to 12
        run: |
          python -m pytest -q \
            tests/test_day08.py tests/test_day09.py \
            tests/test_day10_config.py tests/test_day10_jetson_route.py \
            tests/test_day10_monitoring.py tests/test_day10_telemetry.py \
            tests/test_day11_cloud.py tests/test_day12_pipeline.py
      - name: Track only approved files
        run: python scripts/12_ci_verify.py
      - name: Validate combined Docker Compose
        run: |
          APP_COMMIT_SHA="$GITHUB_SHA" bash scripts/12_dc.sh config --quiet
          APP_COMMIT_SHA="$GITHUB_SHA" bash scripts/12_dc.sh config --services
      - name: Build CPU FastAPI and worker image
        run: docker build --file Dockerfile --tag subject13-platform:day12-ci .

  deploy:
    name: Deploy commit on EC2 using SSM
    needs: ci
    if: ${{ github.ref == 'refs/heads/main' && vars.DAY12_DEPLOY_ENABLED == 'true' }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Checkout the tested commit
        uses: actions/checkout@v4
      - name: Configure AWS via OIDC
        uses: aws-actions/configure-aws-credentials@v6
        with:
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
      - name: Verify AWS role and EC2 managed instance
        env:
          EC2_ID: ${{ vars.AWS_EC2_INSTANCE_ID }}
        run: |
          aws sts get-caller-identity --query Account --output text
          aws ssm describe-instance-information --query "InstanceInformationList[?InstanceId=='${EC2_ID}'].[InstanceId,PingStatus]" --output table
      - name: Deploy and verify the tested Git SHA
        env:
          EC2_ID: ${{ vars.AWS_EC2_INSTANCE_ID }}
        run: |
          python scripts/12_ssm_deploy.py --instance-id "$EC2_ID" --sha "$GITHUB_SHA"
```

**코드 이해하기**

`needs: ci`이므로 CI 실패 시 CD가 시작되지 않습니다. `DAY12_DEPLOY_ENABLED`가 아직 `true`가 아니면 AWS 역할과 EC2를 설정하기 전에 배포하지 않습니다. `id-token: write`는 OIDC 인증용으로 배포 작업에만 부여했습니다.

**오늘의 CI 검증 범위:** GPU와 실제 카메라가 없어도 확인할 수 있는 8~12일차 Python 테스트, Compose 구성, FastAPI·Worker CPU 이미지 빌드입니다. **실제 GPU Docker 이미지는 EC2에서 빌드**하며, CI가 통과했다고 YOLO CUDA 실행까지 검증된 것은 아닙니다.

### 실습 14. CI의 배포 규격 자동 테스트를 작성합니다

아래 파일은 실제 AWS 계정이 없어도 버전 API와 Git SHA 형식, Workflow가 CI에 의존하는지, SSM 실행 실패를 정확히 오류로 표시하는지 검사합니다.

**파일: `tests/test_day12_pipeline.py`**

```python
"""12일차: 실제 AWS·Docker 없이 CI/CD 구성과 배포 계약을 검증합니다."""
import importlib.util
import json
import subprocess
from pathlib import Path

import pytest
import yaml
from fastapi import FastAPI
from fastapi.testclient import TestClient

from app.version_api import current_release, router

ROOT = Path(__file__).resolve().parents[1]


def load_script(name: str):
    path = ROOT / "scripts" / name
    spec = importlib.util.spec_from_file_location(name.removesuffix('.py'), path)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    return module


def test_release_api_when_not_deployed(monkeypatch):
    monkeypatch.delenv("APP_COMMIT_SHA", raising=False)
    app = FastAPI()
    app.include_router(router)
    data = TestClient(app).get("/api/v1/deploy/version").json()
    assert data == {"commit": "unknown", "source": "local", "service": "subject13-platform"}


def test_release_api_from_sha(monkeypatch):
    sha = "b" * 40
    monkeypatch.setenv("APP_COMMIT_SHA", sha)
    assert current_release()["commit"] == sha
    assert current_release()["source"] == "git"


@pytest.mark.parametrize("bad", ["short", "g"*40, "", "1234", "a"*41])
def test_invalid_release_sha_returns_unknown(monkeypatch, bad):
    monkeypatch.setenv("APP_COMMIT_SHA", bad)
    assert current_release()["commit"] == "unknown"


def test_day12_overlay_only_adds_immutable_version_env():
    data = yaml.safe_load((ROOT / "compose.deploy.yaml").read_text(encoding="utf-8"))
    assert set(data["services"]) == {"api", "cloud-gpu"}
    assert "db" not in data["services"] and "redis" not in data["services"]
    assert data["services"]["api"]["environment"]["APP_COMMIT_SHA"]
    assert "volumes" not in data


def test_day12_compose_script_preserves_day11_file_order():
    script = (ROOT / "scripts/12_dc.sh").read_text(encoding="utf-8")
    configs = ["compose.yaml", "compose.monitoring.yaml", "compose.cloud.yaml",
               "compose.cloud.monitoring.yaml", "compose.deploy.yaml"]
    assert [script.index(cfg) for cfg in configs] == sorted(script.index(cfg) for cfg in configs)
    assert "-p subject13" not in script


def test_ci_workflow_gates_deploy_after_tests():
    wf = yaml.safe_load((ROOT / ".github/workflows/day12-ci-cd.yml").read_text(encoding="utf-8"))
    assert wf["jobs"]["deploy"]["needs"] == "ci"
    assert "DAY12_DEPLOY_ENABLED" in wf["jobs"]["deploy"]["if"]
    assert wf["jobs"]["deploy"]["permissions"]["id-token"] == "write"
    assert "cancel-in-progress" in wf["concurrency"]
    steps = wf["jobs"]["ci"]["steps"]
    assert any("pytest" in step.get("run", "") for step in steps)
    assert any("docker build" in step.get("run", "") for step in steps)


def test_release_checker_accepts_real_shapes(monkeypatch):
    mod = load_script("12_release_check.py")
    sha = "a" * 40
    results = {
        "/health": {"status": "ok", "data_ready": True},
        "/api/v1/deploy/version": {"commit": sha},
        "/api/v1/cloud/health": {"connected": True, "runtime": "pytorch_cuda"},
        "/api/v1/cloud/events?limit=1": [],
    }
    monkeypatch.setattr(mod, "read_json", lambda url: results[url.split("http://localhost:8000")[-1]])
    mod.check_release("http://localhost:8000", sha, 1, 0)


@pytest.mark.parametrize("changed", [
    ("/api/v1/deploy/version", {"commit": "b"*40}),
    ("/api/v1/cloud/health", {"connected": False, "runtime": "pytorch_cuda"}),
    ("/api/v1/cloud/health", {"connected": True, "runtime": "cpu"}),
    ("/health", {"status": "failed"}),
])
def test_release_checker_rejects_unhealthy(monkeypatch, changed):
    mod = load_script("12_release_check.py")
    sha = "a"*40
    results = {
        "/health": {"status": "ok"},
        "/api/v1/deploy/version": {"commit": sha},
        "/api/v1/cloud/health": {"connected": True, "runtime": "pytorch_cuda"},
        "/api/v1/cloud/events?limit=1": [],
    }
    results[changed[0]] = changed[1]
    monkeypatch.setattr(mod, "read_json", lambda url: results[url.split("http://localhost:8000")[-1]])
    with pytest.raises(RuntimeError, match="배포 확인 실패"):
        mod.check_release("http://localhost:8000", sha, 1, 0)


def test_ssm_rejects_invalid_sha_before_aws():
    mod = load_script("12_ssm_deploy.py")
    with pytest.raises(ValueError):
        mod.deploy_via_ssm("i-0123456789abcdef0", "not-a-sha")


def test_ssm_rejects_invalid_instance_before_aws():
    mod = load_script("12_ssm_deploy.py")
    with pytest.raises(ValueError):
        mod.deploy_via_ssm("not-an-instance", "a"*40)


def test_ssm_dispatch_builds_pinned_command(monkeypatch):
    mod = load_script("12_ssm_deploy.py")
    called = []
    def fake_aws(*args):
        called.append(args)
        if args[1] == "send-command":
            assert "--parameters" in args
            payload = json.loads(Path(args[args.index("--parameters") + 1][7:]).read_text())
            assert payload["commands"] == [f"sudo -u ubuntu -H /bin/bash {mod.REMOTE_RUNNER} {'a'*40}"]
            return {"Command": {"CommandId": "abc-test"}}
        return {"Status": "Success", "ResponseCode": 0, "StandardOutputContent": "DEPLOY_OK"}
    monkeypatch.setattr(mod, "run_aws", fake_aws)
    assert mod.deploy_via_ssm("i-0123456789abcdef0", "a"*40)["Status"] == "Success"
    assert len(called) == 2


def test_ssm_marks_remote_failure(monkeypatch):
    mod = load_script("12_ssm_deploy.py")
    def fake_aws(*args):
        if args[1] == "send-command":
            return {"Command": {"CommandId": "abc-test"}}
        return {"Status": "Failed", "ResponseCode": 1, "StandardErrorContent": "ROLLBACK_OK"}
    monkeypatch.setattr(mod, "run_aws", fake_aws)
    with pytest.raises(RuntimeError, match="배포 실패"):
        mod.deploy_via_ssm("i-0123456789abcdef0", "a"*40)


def test_remote_deploy_script_syntax_and_safety():
    file = ROOT / "deploy/day12/remote_deploy.sh"
    subprocess.run(["bash", "-n", str(file)], check=True)
    source = file.read_text()
    assert "pg_dump" in source and "flock -n" in source
    assert "rollback()" in source and "git checkout --detach" in source
    assert "--no-build" in source and "scripts/11_dc.sh" in source
    assert "down -v" not in source and "docker volume rm" not in source
    assert "git merge-base --is-ancestor" in source


def test_ci_required_files_present():
    mod = load_script("12_ci_verify.py")
    mod.verify_layout(ROOT)
```

테스트 중 사용하는 AWS 명령 응답은 코드로 만든 **시험 응답**입니다. 이 테스트 자체가 실제 AWS EC2에 접속하거나 새로운 GPU 인스턴스를 생성하지 않습니다.

### 실습 15. 임시 Git 저장소로 Rollback을 직접 검증합니다

실제 운영 EC2에 고장 난 소스를 올리는 대신, **테스트용 임시 Git 저장소에 구버전·신버전을 생성**합니다. 새 버전 빌드가 실패하면 실제로 이전 커밋으로 돌아가고 `.env`가 유지되는지 확인합니다.

**의사코드**

```text
임시 Git 원격 저장소 만들기
        ↓
이전 정상 커밋 생성
        ↓
신규 배포 커밋 생성
        ↓
신규 배포 성공 시나리오 실행
        ↓
신규 빌드 실패 시나리오 실행
        ↓
이전 Git SHA·.env 보존 확인
```

**파일: `tests/test_day12_remote_rollback.py`**

```python
"""운영 서버가 없는 상태에서도 커밋 배포·실패 Rollback을 실제 Git 저장소로 검증."""
import os
import subprocess
import threading
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from pathlib import Path

import pytest

ROOT = Path(__file__).resolve().parents[1]
REMOTE = ROOT / "deploy/day12/remote_deploy.sh"


class HealthyHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/health":
            body = b'{"status":"ok"}'
        elif self.path == "/api/v1/cloud/health":
            body = b'{"connected":true,"runtime":"pytorch_cuda"}'
        else:
            self.send_response(404)
            self.end_headers()
            return
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def log_message(self, *args):
        pass


def git(folder: Path, *args):
    return subprocess.run(["git", *args], cwd=folder, check=True,
                          capture_output=True, text=True).stdout.strip()


def setup_repo(tmp_path: Path):
    # 원격 bare repo와 11일차/12일차 코드 커밋을 실제로 구성합니다.
    origin = tmp_path / "origin.git"
    source = tmp_path / "source"
    instance = tmp_path / "instance"
    source.mkdir()
    git(source, "init", "-b", "main")
    git(source, "config", "user.name", "student")
    git(source, "config", "user.email", "student@example.invalid")
    (source / "scripts").mkdir()
    (source / ".gitignore").write_text(".env\nartifacts/\n", encoding="utf-8")
    (source / "scripts/11_dc.sh").write_text(
        "#!/bin/bash\nset -e\n"
        "if [[ $1 == exec ]]; then printf 'PGDMPfake-backup'; exit 0; fi\nexit 0\n", encoding="utf-8")
    git(source, "add", ".")
    git(source, "commit", "-m", "11th version")
    old = git(source, "rev-parse", "HEAD")
    git(source, "init", "--bare", str(origin))
    git(source, "remote", "add", "origin", str(origin))
    git(source, "push", "-u", "origin", "main")
    subprocess.run(["git", "clone", "-q", "-b", "main", str(origin), str(instance)], check=True)
    (source / "scripts/12_dc.sh").write_text(
        "#!/bin/bash\nset -e\n"
        "if [[ ${SIM_FAIL_BUILD:-0} == 1 && $1 == build ]]; then exit 42; fi\n"
        "exit 0\n", encoding="utf-8")
    (source / "scripts/11_check_cloud_inputs.py").write_text("print('inputs ready')\n", encoding="utf-8")
    (source / "scripts/12_release_check.py").write_text("print('mock health ok')\n", encoding="utf-8")
    git(source, "add", ".")
    git(source, "commit", "-m", "12th version")
    new = git(source, "rev-parse", "HEAD")
    git(source, "push", "origin", "main")
    (instance / ".env").write_text("CI_TEST=true\n", encoding="utf-8")
    model = instance / "artifacts/day05/jetson/yolo11n-pose.pt"
    model.parent.mkdir(parents=True)
    model.write_bytes(b"model")
    return instance, old, new


@pytest.fixture
def environment(tmp_path):
    instance, old, new = setup_repo(tmp_path)
    server = ThreadingHTTPServer(("127.0.0.1", 0), HealthyHandler)
    thread = threading.Thread(target=server.serve_forever, daemon=True)
    thread.start()
    env = os.environ.copy()
    env.update({
        "SUBJECT13_ALLOW_TEST_USER": "1", "SUBJECT13_APP_ROOT": str(instance),
        "SUBJECT13_DEPLOY_STATE": str(tmp_path / "ops"),
        "SUBJECT13_HEALTH_BASE": f"http://127.0.0.1:{server.server_address[1]}",
    })
    yield instance, old, new, env
    server.shutdown()
    server.server_close()
    thread.join(timeout=2)


def test_real_git_commit_deploy_success(environment):
    instance, old, new, env = environment
    completed = subprocess.run(["bash", str(REMOTE), new], cwd=instance,
                               env=env, text=True, capture_output=True, timeout=20)
    assert completed.returncode == 0, completed.stdout + completed.stderr
    assert git(instance, "rev-parse", "HEAD") == new
    assert (Path(env["SUBJECT13_DEPLOY_STATE"]) / "last_good_sha").read_text().strip() == new
    assert list((Path(env["SUBJECT13_DEPLOY_STATE"]) / "backups").glob("*.dump"))
    assert (instance / ".env").exists()


def test_real_git_rollback_after_new_build_failure(environment):
    instance, old, new, env = environment
    env["SIM_FAIL_BUILD"] = "1"
    completed = subprocess.run(["bash", str(REMOTE), new], cwd=instance,
                               env=env, text=True, capture_output=True, timeout=20)
    assert completed.returncode != 0
    assert "ROLLBACK_OK" in completed.stderr, completed.stdout + completed.stderr
    assert git(instance, "rev-parse", "HEAD") == old
    assert not (Path(env["SUBJECT13_DEPLOY_STATE"]) / "last_good_sha").exists()
    assert (instance / ".env").exists()
```

이 테스트는 `tmp_path`의 독립된 폴더에서만 실행되므로 기존 11일차 AWS 프로젝트를 변경하지 않습니다.

### 실습 16. 자동 테스트를 실행합니다

기존 `.venv`에 PyYAML이 없다면 설치합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
source .venv/bin/activate
python -m pip install pyyaml
```

8~12일차의 장비 없이 실행 가능한 테스트를 실행합니다.

```bash
DATABASE_URL='sqlite+pysqlite:///:memory:' python -m pytest -q   tests/test_day08.py tests/test_day09.py   tests/test_day10_config.py tests/test_day10_jetson_route.py   tests/test_day10_monitoring.py tests/test_day10_telemetry.py   tests/test_day11_cloud.py tests/test_day12_pipeline.py   tests/test_day12_remote_rollback.py
```

**현재 제공된 검증용 코드에서 확인한 결과:**

```text
........................................................................ [ 85%]
............                                                             [100%]
84 passed
```

학생마다 기존 프로젝트의 이전 테스트 파일과 패키지 버전이 다르면 출력 개수가 다를 수 있습니다. 중요한 기준은 **실행한 테스트가 모두 통과하는 것**입니다. 이 결과는 테스트용 SQLite와 모의 SSM 응답·임시 Git 저장소를 포함하며, 실제 AWS GPU 통합 실행을 완료했다는 뜻은 아닙니다.

### 실습 17. 코드와 설정을 최종 검사합니다

```bash
python -m compileall -q app scripts
bash -n scripts/11_dc.sh
bash -n scripts/12_dc.sh
bash -n deploy/day12/remote_deploy.sh
python scripts/12_ci_verify.py
bash scripts/12_dc.sh config --quiet
```

`12_ci_verify.py`에서 Git에 추적된 비밀키나 모델 파일을 발견했다면 오류를 해결하기 전까지 배포를 진행하지 않습니다.

---

## PART 5. 첫 CI 실행과 AWS 배포 환경을 준비합니다

### 실습 18. `.gitignore`에 배포 제외 파일을 확인합니다

기존 `.gitignore`를 지우지 말고 다음 항목이 포함되어 있는지 확인합니다. 이미 작성되어 있는 규칙은 중복해서 추가하지 않아도 됩니다.

**파일: `.gitignore` — 확인·추가할 내용**

```gitignore
.env
.env.*
*.pem
*.key
*.dump
backups/
artifacts/
handoff/records/
handoff/provided/
.venv/
__pycache__/
.pytest_cache/
```

다음 명령으로 Git이 실제로 추적 중인 파일을 확인합니다.

```bash
git ls-files | grep -E '(\.env$|\.pem$|\.key$|\.dump$|\.pt$|\.onnx$)' || true
git status
```

민감 파일이 이미 Git에 추적되고 있다면 `.gitignore`만 고쳐도 사라지지 않습니다. 관련 배포를 중단하고 기관 보안 규정에 따라 Git 기록과 인증키 등을 조치합니다.

### 실습 19. 12일차 신규 코드를 Git에 기록합니다

**기존 PC WSL 터미널:**

```bash
cd ~/ai_vision/subject13_ai_platform
git status

git add app/main.py app/version_api.py compose.deploy.yaml   scripts/12_dc.sh scripts/12_release_check.py scripts/12_ci_verify.py   scripts/12_ssm_deploy.py deploy/day12/remote_deploy.sh   .github/workflows/day12-ci-cd.yml   tests/test_day12_pipeline.py tests/test_day12_remote_rollback.py .gitignore

git status
```

**스테이징 목록에 `.env`, 모델 가중치, MP4·JPG 원본, DB 백업, `.pem`이 없음을 확인한 다음** 커밋합니다.

```bash
git commit -m "교과13 12일차: CI CD 자동배포와 Rollback 구현"
git rev-parse HEAD
git status
```

### 실습 20. 첫 GitHub Actions는 CI만 실행합니다

기관에서 외부 GitHub 사용을 허용한 경우에만 다음 명령을 실행합니다.

```bash
git push origin main
```

GitHub 저장소의 **Actions → Subject13 Day12 CI CD → 가장 최근 실행**을 확인합니다. 현재 `DAY12_DEPLOY_ENABLED`를 설정하지 않았거나 `false`이므로 **CI 테스트는 실행되지만 EC2 배포는 건너뛰어야 정상**입니다.

CI의 로그에서는 다음 과정을 확인합니다.

```text
Python 패키지 설치
      ↓
8~12일차 자동 테스트
      ↓
Git 추적 금지 파일 검사
      ↓
기존 Compose + day12 Overlay 검사
      ↓
FastAPI/Worker CPU 이미지 빌드
      ↓
CI Success
```

CI가 실패했다면 먼저 실패한 테스트 또는 Docker 빌드를 고친 뒤 다시 푸시합니다. CI를 통과하지 않은 SHA를 운영 EC2에 수동으로 복사해서 성공으로 처리하지 않습니다.

### 실습 21. EC2의 SSM 연결 상태를 확인합니다

**AWS 콘솔 → EC2**에서 11일차에 사용한 실제 GPU 인스턴스의 **Instance ID와 AWS Region**을 확인합니다. SSM을 사용하려면 **EC2 자체에 SSM Agent와 인스턴스 IAM 역할**이 준비되어 있어야 합니다.

AWS 콘솔의 **Systems Manager → Fleet Manager / Managed nodes**에서 해당 인스턴스가 `Online`인지 확인합니다. 역할에는 일반적으로 `AmazonSSMManagedInstanceCore` 관리형 정책을 사용합니다. 현재 계정에 필요한 설정을 변경할 권한이 없다면 승인된 IAM 담당자와 확인합니다.

**EC2 SSH 터미널**에서는 다음 명령을 사용할 수 있습니다.

```bash
whoami
uname -m
sudo systemctl status amazon-ssm-agent --no-pager || true
sudo snap services amazon-ssm-agent 2>/dev/null || true
```

Ubuntu AMI 구성에 따라 SSM Agent가 Snap으로 설치되어 있을 수 있습니다. 서비스 명령 일부가 실패했다고 즉시 삭제·재설치하지 말고 콘솔의 관리 노드 상태를 우선 확인합니다.

**SSM이 `Offline`이면 배포를 활성화하지 않습니다.** EC2 IAM 역할, SSM 서비스로 나가는 HTTPS 네트워크 경로와 Agent 실행 여부를 점검하세요. 기존 EC2의 `8000`, `3000`, `9090`, `8100`, `5432`, `6379` 포트를 인터넷에 공개하지 않습니다.

### 실습 22. EC2가 승인된 Git 저장소를 읽도록 준비합니다

11일차에는 EC2로 코드를 `rsync`하면서 `.git/` 폴더를 제외했습니다. 이번에는 **기존 파일을 삭제하지 않고** Git 이력을 연결해야 합니다.

GitHub 저장소가 **비공개**라면 EC2에 읽기 전용 Deploy Key를 생성합니다.

**EC2 SSH 터미널:**

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/subject13_readonly -C "subject13-ec2-readonly"
chmod 600 ~/.ssh/subject13_readonly
cat ~/.ssh/subject13_readonly.pub
```

생성된 **공개키(`.pub`)만** GitHub 저장소의 **Settings → Deploy keys → Add deploy key**에 등록합니다. `Allow write access`는 선택하지 않습니다. 개인키는 공개하거나 Git에 올리지 않습니다.

GitHub 서버의 SSH 호스트 키 지문은 [GitHub 공식 지문 안내](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)와 비교해 확인합니다. 지문을 검증하지 않은 상태에서 무조건 신뢰하지 않습니다.

**EC2의 `~/.ssh/config` 파일**에 다음을 추가합니다.

```sshconfig
Host github-subject13
    HostName github.com
    User git
    IdentityFile ~/.ssh/subject13_readonly
    IdentitiesOnly yes
```

```bash
chmod 600 ~/.ssh/config
ssh -T git@github-subject13
```

GitHub가 셸 접근은 제공하지 않는다는 메시지가 나와도 **SSH 인증에 성공했는지** 확인합니다.

> GitHub가 금지된 폐쇄망 강의실에서는 위의 외부 GitHub 설정을 진행하지 않습니다. 내부 GitLab/Gitea 접근은 기관에서 승인한 별도 CI 설정을 사용해야 합니다.

### 실습 23. EC2의 11일차 파일을 보존하면서 Git 이력을 연결합니다

**EC2 터미널**에서 실행합니다. 아래 URL의 `OWNER/REPOSITORY`를 실제 승인된 저장소명으로 바꿉니다. `DAY11_SHA`에는 PART 1에서 기록한 **실제 11일차 커밋**을 적습니다.

```bash
cd ~/ai_vision/subject13_ai_platform
pwd

# 11일차 rsync에는 .git이 없으므로, 없는 경우에만 초기화합니다.
if [ ! -d .git ]; then git init; fi

git remote -v
# origin이 없는 경우에만 추가합니다.
if ! git remote get-url origin >/dev/null 2>&1; then
  git remote add origin git@github-subject13:OWNER/REPOSITORY.git
fi

git fetch origin main

export DAY11_SHA="여기에_기록한_실제_40자리_11일차_SHA"
git cat-file -t "$DAY11_SHA"
git reset --mixed "$DAY11_SHA"
git status --short
```

`git remote add origin`에서 `remote origin already exists`가 나온다면 무조건 추가하지 않고 `git remote -v`로 실제 설정을 확인합니다.

**중요:** `git reset --mixed`는 Git 기준을 11일차 커밋에 맞추되 기존 작업 파일을 지우지 않습니다. `.env`, 모델, DB 데이터도 유지합니다. `git status --short`에 **추적된 소스 파일의 수정 내용이 나타나면** 그 내용을 먼저 비교·보존하고, 소스가 11일차 커밋과 일치할 때까지 배포를 진행하지 않습니다. `git reset --hard`, `git clean -fdx`로 강제로 지우지 않습니다.

이어서 기존 서비스가 정상인지 확인합니다.

```bash
git rev-parse HEAD
bash scripts/11_dc.sh ps
curl -fsS http://127.0.0.1:8000/health
curl -fsS http://127.0.0.1:8000/api/v1/cloud/health
ls -l .env artifacts/day05/jetson/yolo11n-pose.pt
```

### 실습 24. EC2에 배포 실행기를 한 번 설치합니다

신규 버전 코드와 배포 실행기를 같은 폴더에 넣으면 잘못된 버전으로 체크아웃하는 순간 복구 프로그램도 바뀔 수 있습니다. 따라서 **Git 프로젝트 바깥**의 `~/subject13-deploy/`에 배포 실행기를 설치합니다.

**EC2 터미널:**

```bash
mkdir -p "$HOME/subject13-deploy/state/backups"
chmod 700 "$HOME/subject13-deploy" "$HOME/subject13-deploy/state"

# 현재 원격 main의 12일차 배포 실행기 내용만 가져옵니다.
git show origin/main:deploy/day12/remote_deploy.sh   > "$HOME/subject13-deploy/remote_deploy.sh"

chmod 700 "$HOME/subject13-deploy/remote_deploy.sh"
bash -n "$HOME/subject13-deploy/remote_deploy.sh"
ls -l "$HOME/subject13-deploy/remote_deploy.sh"
git rev-parse HEAD
```

Git HEAD는 여전히 **11일차 SHA**여야 합니다. 12일차의 자동 배포를 시작하면 스크립트가 새 SHA로 체크아웃하게 됩니다. `~/subject13-deploy/` 안의 운영 파일을 GitHub에 올리지 않습니다.

### 실습 25. GitHub Actions와 AWS의 OIDC 인증을 설정합니다

AWS IAM에 GitHub OIDC 공급자가 이미 등록되어 있다면 기존 제공자를 사용합니다. 없다면 승인된 관리 권한으로 IAM의 **Identity provider**를 등록합니다.

```text
Provider URL: https://token.actions.githubusercontent.com
Audience:     sts.amazonaws.com
```

GitHub Actions 전용 IAM Role을 만들고, 승인된 저장소 **`main` 브랜치만** 해당 역할을 사용할 수 있도록 Trust Policy를 제한합니다.

**AWS IAM 신뢰 정책 예시:** 아래의 `ACCOUNT_ID`, `OWNER`, `OWNER_ID`, `REPOSITORY`, `REPOSITORY_ID`는 반드시 **실제 AWS 계정과 GitHub 저장소 정보로 교체**합니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:OWNER@OWNER_ID/REPOSITORY@REPOSITORY_ID:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**2026년 7월 15일 이후 생성된 GitHub 저장소**는 OIDC `sub`에 Immutable ID가 포함된 형식을 기본으로 사용합니다. 그 이전에 생성한 저장소 중 새 형식을 선택하지 않은 경우 `repo:OWNER/REPOSITORY:ref:refs/heads/main` 형식일 수 있습니다. GitHub 공식 문서에서 본인 저장소의 실제 Subject Claim을 확인해 정확하게 입력해야 합니다. 무분별하게 `*`로 모든 저장소를 허용하지 않습니다.

이 IAM Role에는 특정 EC2 인스턴스에 SSM 명령을 보내는 권한만 부여합니다. 아래 정책의 `REGION`, `ACCOUNT_ID`, `INSTANCE_ID`를 모두 실제 값으로 교체합니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RunOnApprovedInstanceOnly",
      "Effect": "Allow",
      "Action": "ssm:SendCommand",
      "Resource": [
        "arn:aws:ssm:REGION::document/AWS-RunShellScript",
        "arn:aws:ec2:REGION:ACCOUNT_ID:instance/INSTANCE_ID"
      ]
    },
    {
      "Sid": "ReadRunCommandStatus",
      "Effect": "Allow",
      "Action": ["ssm:GetCommandInvocation", "ssm:DescribeInstanceInformation"],
      "Resource": "*"
    }
  ]
}
```

기관의 보안 정책에 따라 추가 조건이나 IAM 승인이 필요할 수 있습니다. 위 역할은 **SSM을 통해 특정 서버에서 셸 명령을 실행할 수 있는 배포 권한**이므로 별도 관리해야 합니다.

GitHub 공식 AWS OIDC 안내: <https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws>

### 실습 26. GitHub Actions 변수 네 개를 등록합니다

GitHub 저장소의 **Settings → Secrets and variables → Actions → Variables**에서 아래 값을 등록합니다. 이름이 한 글자라도 다르면 Workflow에서 찾지 못할 수 있습니다.

| 변수 이름 | 입력 값 |
|---|---|
| `AWS_REGION` | 본인 EC2 리전. 예: `ap-northeast-2` |
| `AWS_EC2_INSTANCE_ID` | AWS EC2의 실제 인스턴스 ID (`i-...`) |
| `AWS_DEPLOY_ROLE_ARN` | 앞에서 만든 AWS GitHub OIDC 배포 Role ARN |
| `DAY12_DEPLOY_ENABLED` | 모든 준비가 끝나기 전에는 `false`, 준비 후 `true` |

`.env`나 비밀키를 GitHub Repository Variables에 입력하지 않습니다. 기존 EC2의 비밀번호와 API 키는 보호된 서버 `.env`를 그대로 사용합니다.

### 실습 27. 첫 자동 배포를 진행합니다

SSM Managed Node가 온라인이고, EC2 Git 원격·모델·DB·배포 스크립트가 준비되었다면 **`DAY12_DEPLOY_ENABLED`를 `true`로 변경**합니다.

GitHub에서 **Actions → Subject13 Day12 CI CD → Run workflow → Branch `main` → Run workflow**를 클릭합니다. 이제 아래 흐름이 진행되어야 합니다.

```text
CI Python 테스트
       ↓
CI Docker Compose 검사
       ↓
CI CPU 이미지 빌드
       ↓
AWS OIDC 임시 자격 획득
       ↓
SSM 명령 실행
       ↓
EC2 PostgreSQL 백업
       ↓
새 Git SHA로 체크아웃
       ↓
FastAPI·Cloud GPU 이미지 빌드
       ↓
Docker Compose 전체 서비스 재시작/유지
       ↓
Git SHA·API·Cloud GPU·DB 조회 확인
       ↓
DEPLOY_OK
```

**실제 EC2 GPU 이미지 빌드는 시간이 걸릴 수 있습니다.** `deploy` Job이 실패했다면 EC2 배포가 성공했다고 표시하지 말고 SSM 실행 결과와 복구 상태를 확인합니다.

---

## PART 6. AWS EC2에서 실제 배포 결과를 확인합니다

### 실습 28. EC2의 Git SHA와 API 응답을 비교합니다

**EC2 SSH 터미널:**

```bash
cd ~/ai_vision/subject13_ai_platform
git rev-parse HEAD
bash scripts/12_dc.sh ps
curl -fsS http://127.0.0.1:8000/api/v1/deploy/version
```

출력된 JSON의 형식은 다음과 같습니다.

```json
{
  "commit": "실제로_배포된_40자리_SHA",
  "source": "git",
  "service": "subject13-platform"
}
```

위의 문자열은 **구조를 설명하는 예시**입니다. 실제 응답은 40자리 16진수 SHA여야 합니다. `GitHub Actions의 검사 SHA = EC2 git rev-parse HEAD = API commit` 세 값이 반드시 같아야 합니다.

### 실습 29. 기존 Cloud GPU와 PostgreSQL 기록을 확인합니다

**EC2 터미널:**

```bash
curl -fsS http://127.0.0.1:8000/health
curl -fsS http://127.0.0.1:8000/api/v1/cloud/health
curl -fsS 'http://127.0.0.1:8000/api/v1/cloud/events?limit=1'

bash scripts/12_dc.sh exec -T db   sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT COUNT(*) FROM inference_events;"'
```

`/api/v1/cloud/health`의 `connected`가 `true`, `runtime`이 `pytorch_cuda`인지 확인합니다. 배포 전에 메모한 DB 건수와 비교하여 기존 기록이 사라지지 않았는지 확인합니다. 추가 추론을 실행한 경우 건수는 늘어날 수 있습니다.

### 실습 30. 배포 후 자동 검사 프로그램을 직접 실행합니다

**EC2 터미널:**

```bash
cd ~/ai_vision/subject13_ai_platform
export DEPLOYED_SHA="$(git rev-parse HEAD)"
python3 scripts/12_release_check.py   --base-url http://127.0.0.1:8000   --expected-sha "$DEPLOYED_SHA"   --attempts 3 --interval 2
```

정상 출력 예:

```text
PASS: FastAPI + 배포 SHA + Cloud GPU + DB 조회
배포된 커밋: (실제 Git SHA)
```

이것은 API·GPU 준비 상태·DB 조회 검사입니다. **새 이미지를 실제로 추론하는 성능 확인은 다음 실습에서 수행**합니다.

### 실습 31. Cloud GPU의 기존 Pose 추론이 계속 실행되는지 확인합니다

11일차와 마찬가지로, **허가된 JPG/PNG 테스트 이미지**를 웹 화면에서 업로드합니다. EC2의 FastAPI와 Grafana는 외부 인터넷에 공개하지 않고 SSH 터널로 확인합니다.

**기존 개발 PC의 새 WSL 터미널:**

```bash
export EC2_PUBLIC_IP="본인_EC2의_실제_공인_IP"
export SSH_KEY="$HOME/.ssh/subject13-day11.pem"
ssh -i "$SSH_KEY" -N   -L 8000:127.0.0.1:8000   -L 3000:127.0.0.1:3000   ubuntu@"$EC2_PUBLIC_IP"
```

웹브라우저에서 `http://127.0.0.1:8000/cloud`를 열고 이미지를 분석합니다. **Bounding Box, COCO17 Keypoint, GPU 추론 결과 및 `cloud:` 접두사 DB 기록**을 확인합니다. 추론 실패 시에는 API 테스트가 통과했다는 이유만으로 성공 처리하지 않습니다.

### 실습 32. Prometheus·Grafana 대시보드가 유지되는지 확인합니다

위 SSH 터널이 연결된 브라우저에서 `http://127.0.0.1:3000`을 열고 10·11일차 대시보드가 그대로 있는지 확인합니다.

**EC2 터미널:**

```bash
bash scripts/12_dc.sh ps
bash scripts/12_dc.sh logs --tail=50 api
bash scripts/12_dc.sh logs --tail=50 cloud-gpu
bash scripts/12_dc.sh logs --tail=50 prometheus
bash scripts/12_dc.sh logs --tail=50 grafana
```

로그를 공유할 때 DB 비밀번호와 토큰이 드러나지 않도록 주의합니다. 기존 Prometheus와 Grafana 데이터가 삭제되거나 새 볼륨으로 분리되었다면 Compose 프로젝트 이름과 Overlay 순서를 점검합니다.

### 실습 33. 배포 전 DB 백업과 마지막 정상 커밋을 확인합니다

**EC2 터미널:**

```bash
ls -lh "$HOME/subject13-deploy/state/backups/"
cat "$HOME/subject13-deploy/state/last_good_sha"
git rev-parse HEAD
```

마지막 정상 배포 SHA가 현재 Git HEAD와 일치하는지 확인합니다. `backups/` 아래에는 `before_<SHA>_<시각>.dump` 형식의 PostgreSQL 백업이 있어야 합니다. 백업은 운영 데이터가 포함될 수 있어 외부 GitHub에 올리지 않습니다.

**참고:** 백업 파일이 생성되었다는 사실만으로 실제 데이터 복원에 성공한다고 보장할 수는 없습니다. 13일차 Acceptance Test에서 백업 검증과 복원 절차를 다시 점검합니다.

---

## PART 7. Rollback 검증과 장애 해결

### 실습 34. 운영 EC2를 고장 내지 않고 Rollback을 검증합니다

**기존 PC의 터미널**에서 실행합니다. `test_day12_remote_rollback.py`는 임시 Git 저장소에서 새 버전의 빌드 실패를 만들고 이전 커밋으로 돌아가는 과정을 실제로 수행합니다.

```bash
DATABASE_URL='sqlite+pysqlite:///:memory:'   python -m pytest -q tests/test_day12_remote_rollback.py -v
```

아래 두 테스트가 통과해야 합니다.

```text
test_real_git_commit_deploy_success                  PASSED
test_real_git_rollback_after_new_build_failure       PASSED
```

이 결과는 **임시 Git과 가짜 Compose 빌드에서 Rollback이 성공했다는 뜻**입니다. 실제 EC2 CUDA 컨테이너의 장애 복구가 완료되었다고 혼동하지 않습니다.

### 실습 35. AWS 배포 실패 원인을 순서대로 확인합니다

`Actions`의 `deploy`가 실패했을 때는 **SSM 명령 상태와 EC2의 Git HEAD, 컨테이너 상태**를 확인합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
git rev-parse HEAD
cat "$HOME/subject13-deploy/state/last_good_sha" 2>/dev/null || true
bash scripts/11_dc.sh ps
ls -lh "$HOME/subject13-deploy/state/backups/"
```

| 현상 | 확인 지점 | 해결 방법 |
|---|---|---|
| `ci` 실패 | Python 테스트·Docker CPU 빌드 | 먼저 코드 문제 수정. CI 실패 SHA는 배포하지 않음 |
| OIDC `AccessDenied` | Trust Policy의 GitHub `sub`·Role ARN | 실제 계정/저장소/브랜치와 일치 여부 확인 |
| EC2 SSM `Offline` | Agent·인스턴스 역할·네트워크 | Managed node가 Online이 될 때까지 대기 |
| Git 인증 오류 | EC2 읽기 전용 Deploy Key | SSH Host·원격 URL·Deploy Key 접근 확인 |
| EC2 미커밋 소스 | `git status` | 기존 수동 변경 보존·코드 일치 확인 |
| PostgreSQL 백업 오류 | DB 컨테이너·디스크 용량 | 백업 검증 성공 전 새 버전 배포 중지 |
| Cloud GPU Build 오류 | CUDA 이미지·디스크·패키지 | 실제 EC2 로그 확인 후 수정 |
| API 배포 SHA 불일치 | `APP_COMMIT_SHA` | 12일차 Overlay·Compose 실행 방식 점검 |
| `ROLLBACK_FAILED` | 구버전 복귀 단계·API/Cloud GPU | 추가 자동 배포 중단, 11일차 복구 및 DB 백업 확인 |

Rollback은 실패한 새 버전을 자동으로 성공 처리하지 않습니다. 이전 정상 버전으로 복귀한 뒤에도 **GitHub Actions의 실패 로그는 유지**합니다.

### 실습 36. 운영 데이터·키·비용에 대한 안전 규칙을 확인합니다

```text
사용하지 않는 명령
- docker compose down -v
- docker volume rm ...
- git clean -fdx
- 운영 DB를 자동 DROP / ALTER하는 검증되지 않은 마이그레이션
- .env / 인증키 / 모델 / 영상 / DB 백업의 GitHub 업로드
- 비밀번호나 인증 토큰을 포함한 배포 로그 공유
```

오늘은 코드와 컨테이너의 복구만 자동화합니다. **DB 스키마 마이그레이션을 실행하거나, 누락된 GPU 모델을 대신 생성하거나, AWS 서비스 포트를 인터넷에 개방하는 수업이 아닙니다.**

---

## PART 8. 운영 기록과 GitHub 업로드를 정리합니다

### 실습 37. 배포 운영 문서를 작성합니다

나중에 장애가 발생해도 Git SHA와 실제 배포 상태를 다시 찾을 수 있도록 운영 기록을 작성합니다.

**파일: `docs/day12_cicd_runbook.md`**

````markdown
# 교과 13 · 12일차 Git CI/CD 운영 기록

## 1. 오늘 사용한 환경
- 승인된 Git 저장소와 `main` 브랜치:
- EC2 Instance ID와 AWS Region:
- 배포 IAM Role(비밀키 제외):
- 현재 사용 중인 Compose 실행기: `scripts/12_dc.sh`
- EC2 배포 실행기: `~/subject13-deploy/remote_deploy.sh`

## 2. 자동 테스트
- GitHub Actions CI 실행 결과:
- 통과한 자동 테스트 수:
- Docker Compose YAML 검사 결과:
- FastAPI/Worker CPU 이미지 빌드 결과:

## 3. 배포 버전
- GitHub Actions가 배포 요청한 SHA:
- EC2 `git rev-parse HEAD`:
- `GET /api/v1/deploy/version`의 SHA:
- EC2 SSM 명령의 성공·실패 상태:

## 4. 서비스 인수 확인
- FastAPI `/health`:
- Cloud GPU `/api/v1/cloud/health`:
- 실제 JPG/PNG YOLO Pose 추론:
- PostgreSQL 배포 전·후 기록 수:
- Grafana 대시보드:

## 5. PostgreSQL 백업과 복구
- 배포 백업 경로:
- 백업 파일 보관 정책:
- 운영 데이터 복원 필요 여부 및 실제 확인 결과:

## 6. Rollback
- 실패한 배포 SHA:
- 실패 단계:
- 복귀한 이전 정상 SHA:
- ROLLBACK_OK 또는 ROLLBACK_FAILED:
- 복귀 후 실제 API/GPU 상태:

## 7. 비용·보안
- `.env`, 모델·DB 백업의 Git 제외 확인:
- EC2 보안그룹 공개 포트 확인:
- GPU EC2 Stop/Terminate 및 스토리지 비용 확인:

## 8. 13일차 연결
- 최종 Acceptance Test에 필요한 Git·Docker·DB·GPU·모니터링·복구 결과:
````

실제로 수행하지 않은 검증은 **`미확인`**으로 남깁니다. 원격 배포 성공 결과나 GPU 측정값을 추측해서 적지 않습니다.

### 실습 38. README와 시스템 구조도를 12일차까지 연결합니다

기존 `README.md`의 마지막에 다음 내용을 추가합니다. 기존 1~11일차 실행 안내는 삭제하지 않습니다.

````markdown
## 12일차 — Git CI/CD · AWS 자동 배포 · Rollback

- Workflow: `.github/workflows/day12-ci-cd.yml`
- 버전 API: `GET /api/v1/deploy/version`
- Compose 실행: `bash scripts/12_dc.sh <command>`
- 로컬 CI 검증: 8~12일차의 장치 비의존 자동 테스트
- 배포: GitHub OIDC → AWS SSM → EC2 GPU
- EC2 실행기: `~/subject13-deploy/remote_deploy.sh`
- 기존 PostgreSQL 자동 백업 및 새 버전 실패 시 코드 Rollback
- 운영 기록: `docs/day12_cicd_runbook.md`
````

기존 `docs/architecture.md`에도 다음 흐름을 추가합니다.

````markdown
## 12일차 — CI/CD 연결 구조

Git main → GitHub Actions CI → Python 테스트·CPU Docker 빌드
                            ↓
                      통과 시에만 배포
                            ↓
                    AWS OIDC → AWS SSM
                            ↓
                 EC2 PostgreSQL 사전 백업
                            ↓
              Git SHA Checkout → CPU/GPU Build
                            ↓
              Docker Compose 시작 → 상태 확인
                            ↓
                  성공 확정 / 실패 Rollback

기존 DB 네임드 볼륨·모델·비밀키는 Git 배포 대상에서 제외합니다.
````

### 실습 39. 소스와 문서를 최종 커밋합니다

**기존 PC WSL 터미널**에서 실행합니다.

```bash
cd ~/ai_vision/subject13_ai_platform
python scripts/12_ci_verify.py

git add README.md docs/architecture.md docs/day12_cicd_runbook.md
git status
```

`git status`에서 `.env`, 모델, SSH 개인키, 백업이 스테이징되지 않았는지 반드시 확인합니다. 확인 후 기록합니다.

```bash
git commit -m "교과13 12일차: CI CD 운영 문서 및 배포 검증 정리"
git log -3 --oneline
git status
```

외부 GitHub 사용이 승인된 경우에만 푸시합니다.

```bash
git push origin main
```

**참고:** `DAY12_DEPLOY_ENABLED=true` 상태에서는 **README만 수정한 커밋도 CI 통과 후 AWS 배포 대상**이 될 수 있습니다. 불필요한 GPU 서버 실행·이미지 재빌드 시간을 줄이려면 배포 일정과 비용을 확인합니다.

---

# 6. 오늘 작성한 내용의 최종 정리 및 GitHub 업로드

## 6.1. 오늘 완성한 결과

오늘은 11일차까지 만들었던 AWS GPU AI 플랫폼을 새로 만드는 대신, **Git에 기록한 변경을 테스트하고, 검증된 커밋을 EC2에 배포하며, 비정상일 때 이전 버전으로 복구하는 방법**을 구현했습니다.

```text
교과 13 실제 AI 플랫폼
FastAPI · PostgreSQL · Cloud GPU
               ↓
          Git 변경 기록
               ↓
        GitHub Actions CI
               ↓
    Python 테스트 + Compose 검사
               ↓
         CPU Docker 빌드
               ↓
       AWS OIDC → SSM
               ↓
      EC2 PostgreSQL 백업
               ↓
        검증된 SHA 배포
               ↓
     실제 API·GPU·DB 상태 확인
               ↓
   배포 성공 / 이전 SHA Rollback
```

자동 테스트에서는 **기존 8~11일차 기능과 12일차 배포 코드**, 실제 임시 Git 저장소의 성공·실패 복귀 과정을 검사했습니다. 실제 AWS 네트워크·SSM·EC2 GPU·PostgreSQL의 통합 실행은 학생이 실제 서비스에서 확인한 값만 운영 기록에 남깁니다.

## 6.2. 최종 자가 체크리스트

- [ ] 11일차의 `subject13_ai_platform` 프로젝트·모델·DB를 그대로 유지했습니다.
- [ ] 현재 배포 커밋을 응답하는 `app/version_api.py`를 추가했습니다.
- [ ] 기존 Router를 삭제하지 않고 `app/main.py`에 새 Router를 연결했습니다.
- [ ] `compose.deploy.yaml`로 기존 네임드 볼륨을 바꾸지 않고 버전 정보만 추가했습니다.
- [ ] Python 테스트·Compose 검사·CPU 이미지 빌드 CI를 구성했습니다.
- [ ] `.env`, 모델, 개인키, DB 백업이 Git 추적에 포함되지 않았습니다.
- [ ] 임시 Git 저장소에서 Rollback 테스트가 통과했습니다.
- [ ] EC2가 이전 11일차 코드와 데이터 그대로 Git 원격을 읽을 수 있습니다.
- [ ] AWS SSM Managed node와 GitHub OIDC 신뢰 정책이 정상입니다. (AWS 실습을 한 경우)
- [ ] GitHub Actions의 배포 SHA와 EC2 Git SHA·API 버전이 일치합니다. (AWS 실습을 한 경우)
- [ ] Cloud GPU·PostgreSQL 기록·Grafana가 정상입니다. (AWS 실습을 한 경우)
- [ ] `docs/day12_cicd_runbook.md`와 README·구조도를 완성했습니다.
- [ ] 오늘 작성한 소스만 Git에 커밋했습니다.

## 6.3. 다음 13일차 수업과 연결합니다

**13일차는 교과 13을 마무리하는 Acceptance Test와 프로젝트 인계 수업입니다.** 오늘의 Git CI/CD뿐 아니라 다음 기능을 **각각 실제로 검증**하고 정상·미확인을 구분해서 기록합니다.

- 교과 9·10의 실제 결과 및 공통 JSON, PostgreSQL 데이터 보존
- Redis·Celery 비동기 처리, API·웹 기록 조회
- Raspberry Pi ONNX CPU·GPIO LED 동작
- Jetson Orin Nano Super 실시간 Pose·TensorRT와 FPS 실측
- AWS EC2 GPU Pose 추론·Grafana 상태
- Git 자동 테스트·배포 SHA 확인·Rollback·장애 복구

하나의 `subject13_ai_platform` 프로젝트를 그대로 이어서 **최종 시스템 구성도, 실행 가이드, 검증 기록 및 인수 문서**를 완성합니다.

---

## 공식 참고 문서

- [GitHub Actions](https://docs.github.com/en/actions/writing-workflows)
- [GitHub Actions와 AWS OIDC](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- [AWS SSM Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/run-command.html)
- [AWS CLI SSM SendCommand](https://docs.aws.amazon.com/cli/latest/reference/ssm/send-command.html)
- [Docker Compose 파일 병합](https://docs.docker.com/compose/how-tos/multiple-compose-files/merge/)
- [PostgreSQL 백업](https://www.postgresql.org/docs/current/backup-dump.html)

