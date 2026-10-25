```
Jetson 전원 OFF
microSD → 주황색 리더기 → Windows PC 연결 완료
JetPack 6.2.1 ZIP 다운로드 완료
Raspberry Pi Imager 설치되어 있음
```

라즈베리 파이 이미저를 열어서 다음 순서대로 설치합니다.

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919183544.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919183803.png]]

제공해준 파일을 압축을 푼후 선택합니다.

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919184231.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919184322.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919193340.png]]

- 주황색 microSD 리더기를 PC에서 빼고, 리더기에서 microSD 카드를 꺼냅니다. 이때 Windows가 나중에 “포맷해야 합니다”라고 물으면 포맷하지 말고 취소하세요.
- Jetson 전원이 완전히 꺼져 있는지 확인합니다. 현재 전원 어댑터가 빠져 있다면 그대로 두면 됩니다. 지금은 Jetson에 전원을 넣지 않습니다.
- 방금 만든 microSD를 Jetson Orin Nano 모듈 밑면의 microSD 슬롯에 삽입합니다. 아까 사진에서 확인했던 방열판 아래쪽의 얇은 슬롯입니다. 방향이 맞으면 무리한 딸각하며 힘 없이 들어갑니다. 억지로 밀지 마세요.
- Jetson에는 다음을 그대로 연결해 둡니다.

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919194058.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919194037.png]]

- 마지막으로 Jetson의 둥근 DC 전원 어댑터를 연결합니다. 초록색 `PWR` LED가 들어오는지 확인합니다. 처음 부팅은 평소보다 오래 걸릴 수 있으므로 몇 분은 기다려 주세요.

전원을 연결하면 다음과 같은 화면이 나옵니다

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919194342.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919194705.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919194655.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919194719.png]]

키보드 레이아웃은 지금처럼 왼쪽에서 **Korean**을 선택하고, 오른쪽에서 가능하면 `Korean - Korean (101/104-key compatible)`를 선택하는 것을 권합니다.

이유는 일반적인 한국 PC 키보드가 101/104키 배열인 경우가 많아서, 한/영 전환이나 특수키 배치가 더 자연스럽게 맞을 가능성이 높기 때문입니다.

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919195359.png]]

![[Pasted image 20260919200434.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919222314.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919200422.png]]

그냥 skip을 누릅니다.

![[Pasted image 20260919200542.png]]

`Skip for now` 그대로 두고 오른쪽 위 `Next`를 누르세요.

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919200641.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919201254.png]]

Ubuntu 초기 설정의 마지막 화면입니다. 지금은 추가 프로그램을 설치하지 말고 오른쪽 위 `Done`을 누르시면 됩니다.

![[Pasted image 20260919201435.png]]

재부팅을 합니다.

```
Power Off / Log Out 또는 전원 끄기 / 로그아웃

Restart 다시 시작
```

재부팅후 설치가 제대로 되었는지 검증합니다.
키보드에서:

```bash
Ctrl + Alt + T
```

를 누르세요.

### Jetson Linux 버전 확인

```bash
cat /etc/nv_tegra_release
```

정상이라면

```
# R36 (release), REVISION: 4.4 ...
```

### 장비 모델 확인

```bash
cat /proc/device-tree/model
```

정상이라면 Jetson Orin Nano 관련 이름이 나옵니다.

### Ubuntu 버전 확인

```bash
cat /etc/os-release
```

우리가 설치한 환경이라면:

```
Ubuntu 22.04
```

계열이 나오는 것이 정상입니다.

JetPack 구성요소는 패키지 저장소에서 설치할 수 있고, 설치 명령으로 아래 두 줄을 안내합니다.

젯슨터미널에서 다음과 같이 설치합니다.

```bash
sudo -k
sudo -v
```

```bash
sudo apt update 
sudo apt install nvidia-jetpack
```

### SSH 연결하기

우선 Jetson 터미널에서

```bash
hostname -I
```

입력을 하면 나오는 주소를 일반 PC 파워쉘에서 다음과 같이 입력합니다.

```powershell
ssh-keygen -R 192.168.219.102
```

`ssh-keygen -R`은 앞으로 매번 하면 안 됩니다. 다음과 같은 경고가 다시 나올 때만 사용합니다.

그러면 대략 이런 식으로 나옵니다.

```
# Host 192.168.219.102 found: line 4
C:\Users\123\.ssh\known_hosts updated.
```

처음 연결처럼 아래가 나오면:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

여기서:

```
yes
```

입력하고 Enter를 누르세요.

그리고 비밀번호를 누르면

```
youjung@192.168.219.102's password:
```

성공하면 PowerShell이 이런 식으로 바뀝니다.

```
youjung@jetson-orin:~$
```

그러면 **Windows PC → Jetson SSH 연결 성공**입니다.

---

### 다시 접속할때

1. Jetson에 전원만 연결해서 켭니다. microSD는 그대로 꽂혀 있어야 합니다. 모니터, 키보드, 마우스는 빼도 됩니다. Jetson이 Ubuntu까지 부팅되고 Wi-Fi에 자동으로 연결될 때까지 1~2분 정도 기다립니다.

2. Windows PC도 Jetson과 같은 공유기에 연결되어 있어야 합니다. 현재 Jetson IP가 `192.168.219.102`이므로, PC도 `192.168.219.xxx` 네트워크에 있으면 됩니다.

3. Windows에서 일반 PowerShell을 실행합니다. 

4. 원하면 먼저 Jetson이 살아 있는지 확인합니다.

```bash
ping 192.168.219.102
```

응답이 오면 네트워크 연결이 된 것입니다.

5. 바로 SSH 접속합니다.

```bash
ssh youjung@192.168.219.102
```

6. Jetson 비밀번호를 입력합니다. 비밀번호를 입력해도 화면에 글자나 `*****`가 보이지 않는 것이 정상입니다.
7. 성공하면:

```bash
youjung@jetson-origin:~$
```

처럼 바뀝니다. 이 순간부터 이 PowerShell 창은 Jetson 터미널입니다.

---

### VSCode WSL2 터미널에서 jetson 열기

Extensions에서 다음 확장을 설치합니다.

```
Remote - SSH
```

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{4E977A3A-CBB8-452C-8D99-8ECCF14FD25F}.png]]

설치가 끝나면 다음 순서로 진행하세요.

```
Ctrl + Shift + P
        ↓
Remote-SSH: Connect to Host...
        ↓
youjung@192.168.219.102
        ↓
Linux 선택
        ↓
Jetson 비밀번호 입력
```

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{AF96FB65-4398-4415-A5DF-955595C66166}.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{25B01640-99B2-406A-9B12-F2911D134BCF}.png]]

```
ssh youjung@192.168.219.102
```

Enter를 누르면 SSH 설정 파일을 어디에 저장할지 물어볼 겁니다. 여러 개가 나오면 Windows 사용자 계정 쪽을 선택하세요. 

보통:

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{564C1A78-8F5C-4677-BF19-52375F155820}.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{3FE9E14E-2950-4415-B032-9E3ED0E012E9}.png]]

그러면 **`Connect`** 를 누르세요.

만약 알림을 놓쳤다면 다시:

```
Ctrl + Shift + P
```

→

```
Remote-SSH: Connect to Host...
```

를 선택하면 이제 목록에:

```
192.168.219.102
```

또는

```
youjung@192.168.219.102
```

가 나타날 겁니다. 그것을 선택하세요.

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{BF11E6AB-1F53-4FE2-B0F2-CAC5B418D4E3}.png]]

처음 연결하면서 운영체제를 묻는다면:

```
Linux
```

를 선택합니다.

그리고 비밀번호를 입력합니다.

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{CB85E523-FFB4-461C-BBDD-18ECFC0F18A6}.png]]

두개를 연결하여 앞으로 사용하면 됩니다.

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260919231536.png]]


---
### Jetson Linux 버전 확인

```bash
cat /etc/nv_tegra_release
```

정상적으로 메타 패키지가 설치되어 있으면 버전이 출력됩니다.

예:

```
# R36 (release), REVISION: 4.4, GCID: 41062509, BOARD: generic, EABI: aarch64, DATE: Mon Jun 16 16:07:13 UTC 2025
# KERNEL_VARIANT: oot
TARGET_USERSPACE_LIB_DIR=nvidia
TARGET_USERSPACE_LIB_DIR_PATH=usr/lib/aarch64-linux-gnu/nvidia
```

### Jetson Orin Nano 관련 모델명 확인

```bash
cat /proc/device-tree/model
```

정상이라면 

```
NVIDIA Jetson Orin Nano Engineering Reference Developer Kit Super
```
### Ubuntu 22.04 계열인지 확인

```bash
cat /etc/os-release
```

결과 확인

```
PRETTY_NAME="Ubuntu 22.04.5 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.5 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```

###  `nvidia-jetpack`을 설치할 수 있는 상태인지 먼저 확인

```bash
apt-cache policy nvidia-jetpack
```

정상이라면:

```
nvidia-jetpack:
  Installed: (none)
  Candidate: 6.2.1+b38
  Version table:
     6.2.1+b38 600
        600 https://repo.download.nvidia.com/jetson/common r36.4/main arm64 Packages
     6.2+b77 600
        600 https://repo.download.nvidia.com/jetson/common r36.4/main arm64 Packages
     6.1+b123 600
        600 https://repo.download.nvidia.com/jetson/common r36.4/main arm64 Packages
```

이렇게 나오면 저장소는 정상이고 아직 설치만 안 된 상태입니다.

### 저장공간 확인

```bash
df -h /
```

버전 숫자가 나오면 정상입니다.

### 9. 저장공간 확인

```bash
df -h
```

그런데 현재:

```
/dev/mmcblk0p1
Size  : 28G
Used  : 21G
Avail : 6.3G
Use%  : 77%
```

입니다. 

---
### JetPack 설치

```bash
sudo apt install nvidia-jetpack
```

현재 여유 공간이 약 `6.3GB`이고 실제 추가 사용량이 약 `518MB`이므로 충분합니다.

`Do you want to continue? [Y/n]`가 나오면 Y를 누르세요.

설치가 끝나서 다시:

```bash
youjung@jetson-origin:~$
```

설치가 되었는지 확인합니다.

```bash
dpkg-query --show nvidia-jetpack
```

이제 정상이라면 대략:

```bash
nvidia-jetpack    6.2.1+b38
```

처럼 나옵니다.

---
CUDA 설치 여부 확인

```bash
/usr/local/cuda/bin/nvcc --version
```

결과 확인

```bash
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2024 NVIDIA Corporation
Built on Wed_Aug_14_10:14:07_PDT_2024
Cuda compilation tools, release 12.6, V12.6.68
Build cuda_12.6.r12.6/compiler.34714021_0
```

CUDA 12.6 자체는 정상 설치되어 있습니다.

---
CUDA 환경변수 등록

```bash
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

`nvcc`와 CUDA 라이브러리를 어느 터미널에서든 바로 찾을 수 있도록 `PATH`와 `LD_LIBRARY_PATH`를 `~/.bashrc`에 등록하고 즉시 적용한 작업입니다.

---
CUDA 환경변수 적용 확인

```bash
nvcc --version
```

결과 확인

```bash
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2024 NVIDIA Corporation
Built on Wed_Aug_14_10:14:07_PDT_2024
Cuda compilation tools, release 12.6, V12.6.68
Build cuda_12.6.r12.6/compiler.34714021_0
```

---
TensorRT 설치 확인

```bash
python3 -c "import tensorrt as trt; print(trt.__version__)"
```

결과

```
10.3.0
```

---
OpenCV 설치 확인

```bash
python3 -c "import cv2; print(cv2.__version__)"
```

결과

```
4.8.0
```

---
cuDNN 설치 확인

```bash
dpkg -l | grep cudnn
```

---
Jetson 전원 모드 확인

```bash
sudo nvpmodel -q
```

결과

```
NV Power Mode: 25W 1
```

현재 Jetson Orin Nano Super의 전원 모드는 **25W 모드**로 설정되어 있습니다.

---
Jetson 저장공간 확인

```bash
df -h /
```

결과

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p1   28G   22G  5.0G  82% /
```