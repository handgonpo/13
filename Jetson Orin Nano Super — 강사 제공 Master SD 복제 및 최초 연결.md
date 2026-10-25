## 이 실습에서는 무엇을 하나요?

교과 14에서는 Jetson Orin Nano Super에서 다음 환경을 사용합니다.

```
Ubuntu 22.04
JetPack 6.2.1
CUDA 12.6
cuDNN 9.3
TensorRT 10.3
OpenCV 4.8
SSH
VSCode Remote SSH
```

각 학생이 이 환경을 처음부터 개별 설치하면 다운로드 속도, 패키지 설치 오류, 버전 차이 등으로 많은 시간이 소요될 수 있습니다.

따라서 수업에서는 강사가 설치와 검증을 완료한 Master Image를 학생 microSD에 복제하여 동일한 환경을 사용합니다.

```
강사 Jetson
설치·검증 완료
        ↓
Master Image 생성
        ↓
학생 microSD에 복제
        ↓
학생 Jetson 부팅
        ↓
학생별 네트워크 설정
        ↓
SSH / VSCode 연결
        ↓
환경 검증
        ↓
교과 14 실습 시작
```

---

# PART 1. 강사가 제공한 Master Image를 확인합니다

강사가 다음과 같은 이미지 파일을 제공합니다.

예:

```
Jetson_Class_Master_JP621.img
```

이 이미지에는 이미 다음 환경이 들어 있습니다.

```
JetPack 6.2.1
CUDA 12.6
cuDNN 9.3
TensorRT 10.3
OpenCV 4.8
CUDA 환경변수
SSH 환경
수업용 사용자 계정
```

따라서 학생은 NVIDIA JetPack을 처음부터 다시 설치하지 않습니다.

---

# PART 2. 학생 microSD를 PC에 연결합니다

microSD 카드를 USB 카드리더기에 넣고 Windows PC에 연결합니다.

Windows에서 다음과 같은 메시지가 나타날 수 있습니다.

```
이 드라이브를 사용하려면 포맷해야 합니다.
```

이 경우:

```
취소
```

를 누릅니다.

Windows에서 임의로 포맷하지 않습니다.

---

# PART 3. Raspberry Pi Imager를 실행합니다

Raspberry Pi Imager를 실행합니다.

운영체제 선택에서:

```
사용자 정의 사용
Use Custom
```

을 선택합니다.

강사가 제공한:

```
Jetson_Class_Master_JP621.img
```

파일을 선택합니다.

---

# PART 4. 학생 microSD를 선택합니다

저장 장치에서 자신의 microSD를 선택합니다.

반드시 **microSD 용량과 드라이브를 확인한 후 선택**합니다.

다른 USB 드라이브나 Windows 시스템 디스크를 선택하지 않도록 주의합니다.

---

# PART 5. Master Image를 학생 SD에 기록합니다

`쓰기`를 선택합니다.

전체 흐름은 다음과 같습니다.

```
Jetson_Class_Master_JP621.img
             ↓
     Raspberry Pi Imager
             ↓
            쓰기
             ↓
       학생 microSD
```

쓰기 후 검증이 완료될 때까지 microSD를 제거하지 않습니다.

완료되면 microSD를 Windows에서 안전하게 제거합니다.

---

# PART 6. 학생 Jetson에 microSD를 장착합니다

Jetson의 전원이 꺼진 상태에서 복제된 microSD를 Jetson Orin Nano Super에 장착합니다.

그다음 Jetson 전원을 연결합니다.

```
복제된 microSD
      ↓
Jetson에 장착
      ↓
전원 ON
      ↓
Ubuntu 자동 부팅
```

이미 초기 설정이 완료된 이미지를 복제했기 때문에 정상적인 경우 Ubuntu 설치 화면부터 다시 진행하지 않습니다.

---

# PART 7. 모니터 없이 Jetson에 연결합니다

Jetson Orin Nano Super의 USB-C 데이터 포트는 **USB Device Mode**를 지원하며 PC와 Jetson 사이에 USB Ethernet을 만들 수 있습니다. 이때 Jetson은 기본적으로 `192.168.55.1` 주소를 사용합니다.

학생 PC와 Jetson을 **데이터 통신이 가능한 USB-C 케이블**로 연결합니다.

```
학생 Windows PC
       │
       │ USB-C Data
       │
       ▼
Jetson Orin Nano Super

Jetson USB 주소
192.168.55.1
```

Jetson에는 별도의 모니터, 키보드, 마우스를 연결하지 않아도 됩니다.

---

# PART 8. Windows PC에서 Jetson 연결을 확인합니다

Windows PowerShell을 실행합니다.

관리자 권한은 필요하지 않습니다.

먼저:

```
ping 192.168.55.1
```

을 실행합니다.

응답이 오면 PC와 Jetson의 USB 네트워크가 정상입니다.

---

# PART 9. SSH로 Jetson에 접속합니다

PowerShell에서:

```
ssh youjung@192.168.55.1
```

을 입력합니다.

처음 연결할 때:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

가 나오면:

```
yes
```

를 입력합니다.

그다음 강사가 알려준 **수업용 공통 비밀번호**를 입력합니다.

비밀번호 입력 중에는 화면에 문자가 표시되지 않는 것이 정상입니다.

성공하면:

```
youjung@jetson-origin:~$
```

처럼 표시됩니다.

이제 Windows PC에서 Jetson 터미널을 사용할 수 있습니다.

---

# PART 10. 복제된 Jetson을 학생별로 구분합니다

Master Image를 복제했으므로 처음에는 모든 Jetson의 hostname이 동일합니다.

학생별로 장비를 쉽게 구분하기 위해 hostname을 변경합니다.

예를 들어 1번 학생:

```
sudo hostnamectl set-hostname jetson-s01
```

2번 학생:

```
sudo hostnamectl set-hostname jetson-s02
```

3번 학생:

```
sudo hostnamectl set-hostname jetson-s03
```

학생 번호에 맞춰 설정합니다.

예:

```
01 → jetson-s01
02 → jetson-s02
03 → jetson-s03
...
30 → jetson-s30
```

---

# PART 11. 복제 장비의 고유 ID와 SSH Key를 새로 만듭니다

Master SD를 그대로 복제하면 여러 Jetson에 동일한 시스템 ID와 SSH Host Key가 복제될 수 있습니다.

여러 대를 같은 교육장 네트워크에서 사용할 것이므로 **각 Jetson마다 한 번만 새로 생성**합니다.

먼저 Machine ID를 새로 생성합니다.

```
sudo truncate -s 0 /etc/machine-id
sudo systemd-machine-id-setup
```

그다음 SSH Host Key를 새로 생성합니다.

```
sudo rm -f /etc/ssh/ssh_host_*
sudo ssh-keygen -A
```

이 작업은 **학생 Jetson 최초 사용 시 한 번만 수행**합니다.

---

# PART 12. 교육장 Wi-Fi에 연결합니다

강사 Master Image는 강사가 설정한 장소의 Wi-Fi 정보가 들어 있을 수 있습니다.

교육장에서는 교육장 Wi-Fi로 변경합니다.

먼저 주변 Wi-Fi를 확인합니다.

```
nmcli dev wifi list
```

교육장 Wi-Fi에 연결합니다.

```
nmcli dev wifi connect "교육장_WIFI_SSID" password "교육장_WIFI_PASSWORD"
```

예:

```
nmcli dev wifi connect "AI-Campus" password "12345678"
```

`AI-Campus`, `12345678` 부분은 실제 교육장 정보로 변경합니다.

---

# PART 13. 교육장에서 받은 IP를 확인합니다

Wi-Fi 연결 후:

```
hostname -I
```

을 실행합니다.

예:

```
192.168.10.115 192.168.55.1 172.17.0.1
```

여기서 각각 의미가 다릅니다.

```
192.168.10.115
→ 교육장 Wi-Fi에서 받은 실제 IP

192.168.55.1
→ PC ↔ Jetson USB 연결용 IP

172.17.0.1
→ Docker 내부 네트워크 IP
```

교육장 Wi-Fi를 통한 SSH에서는:

```
192.168.10.115
```

와 같은 주소를 사용합니다.

---

# PART 14. Windows PC에서 교육장 IP로 다시 연결합니다

예를 들어 학생 Jetson의 IP가:

```
192.168.10.115
```

라면 PowerShell에서:

```
ssh youjung@192.168.10.115
```

을 실행합니다.

비밀번호를 입력합니다.

성공하면:

```
youjung@jetson-s01:~$
```

처럼 표시됩니다.

---

# PART 15. SSH Host Key 경고가 나오는 경우

다음 메시지가 나올 수 있습니다.

```
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

이 경우 해당 IP의 기존 SSH 정보를 삭제합니다.

예:

```
ssh-keygen -R 192.168.10.115
```

그다음 다시:

```
ssh youjung@192.168.10.115
```

을 실행합니다.

`ssh-keygen -R`은 매번 실행하는 명령이 아니라 **Host Key 경고가 발생할 때만 사용**합니다.

---

# PART 16. VSCode Remote SSH로 연결합니다

학생 PC의 VSCode에서:

```
Remote - SSH
```

확장을 설치합니다.

그다음:

```
Ctrl + Shift + P
        ↓
Remote-SSH: Connect to Host...
        ↓
ssh youjung@학생_Jetson_IP
```

예:

```
ssh youjung@192.168.10.115
```

운영체제를 묻는 경우:

```
Linux
```

를 선택합니다.

수업용 비밀번호를 입력합니다.

VSCode 왼쪽 아래에:

```
SSH: 192.168.10.115
```

등이 표시되면 Jetson 연결이 완료된 것입니다.

---

# PART 17. 수업용 공통 비밀번호는 그대로 사용해도 됩니다

교육용 장비이고 폐쇄된 실습 환경에서 사용하는 경우에는 **모든 학생이 동일한 수업용 계정과 공통 비밀번호를 사용해도 실습 자체에는 문제가 없습니다.**

오히려:

```
비밀번호 분실 방지
강사 장애 대응 용이
학생별 sudo 문제 감소
환경 통일
```

이라는 장점이 있습니다.

따라서 교과 14 수업에서는 **공통 계정 + 공통 비밀번호 유지**를 권장합니다.

학생 개인 비밀번호 변경은 필수로 하지 않아도 됩니다.

---

# PART 18. 학생이 비밀번호를 변경하고 싶은 경우

필요한 경우:

```
passwd
```

를 실행합니다.

현재 비밀번호를 입력한 뒤 새로운 비밀번호를 두 번 입력합니다.

단, 수업 중 비밀번호를 변경한 경우 **학생 본인이 반드시 기억해야 합니다.**

강사가 장애 대응을 해야 하는 수업용 장비라면 공통 비밀번호를 유지하는 편이 더 편합니다.

---

# PART 19. JetPack 환경이 제대로 복제되었는지 확인합니다

학생은 최종적으로 다음 명령을 실행합니다.

JetPack:

```
dpkg-query --show nvidia-jetpack
```

정상 예:

```
nvidia-jetpack  6.2.1+b38
```

CUDA:

```
nvcc --version
```

정상 핵심 결과:

```
Cuda compilation tools, release 12.6
```

TensorRT:

```
python3 -c "import tensorrt as trt; print(trt.__version__)"
```

정상 결과:

```
10.3.0
```

OpenCV:

```
python3 -c "import cv2; print(cv2.__version__)"
```

정상 결과:

```
4.8.0
```

cuDNN:

```
dpkg -l | grep cudnn
```

전원 모드:

```
sudo nvpmodel -q
```

정상 예:

```
NV Power Mode: 25W
```

---

# 최종 수업 시작 상태

학생 모두 다음 상태가 되면 교과 14 Jetson 실습을 시작할 수 있습니다.

```
학생용 Master SD 복제
        ↓
Jetson 부팅
        ↓
USB-C 192.168.55.1로 최초 SSH
        ↓
학생별 hostname 설정
        ↓
Machine ID / SSH Key 재생성
        ↓
교육장 Wi-Fi 연결
        ↓
교육장 IP 확인
        ↓
VSCode Remote SSH
        ↓
JetPack 6.2.1 확인
        ↓
CUDA 12.6 확인
        ↓
cuDNN 9.3 확인
        ↓
TensorRT 10.3 확인
        ↓
OpenCV 4.8 확인
        ↓
교과 14 실습 시작
```