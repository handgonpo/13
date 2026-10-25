
### 전체 순서
```
microSD
→ 주황색 USB 리더기에 꽂음
→ Windows PC에 연결됨
→ G: bootfs로 보임

Windows PC
→ Raspberry Pi 사이트 열어둠

Raspberry Pi Imager 설치
        ↓
SD카드 새로 설치
        ↓
Wi-Fi + SSH 설정
        ↓
SD카드를 Raspberry Pi에 장착
        ↓
Raspberry Pi 전원 ON
        ↓
집 Wi-Fi 연결
        ↓
WSL2에서 SSH 접속
```

Windows 탐색기에서 microSD의 파일을 하나씩 삭제하거나 직접 포맷할 필요는 없습니다. Raspberry Pi Imager가 기존 내용을 삭제하고 새 운영체제로 다시 구성합니다. 기존 자료가 필요한 경우에는 기록 전에 반드시 백업합니다.
```
기존 SD 카드
       ↓
Raspberry Pi Imager
       ↓
전체를 새 Raspberry Pi OS로 덮어쓰기
```
Imager가 저장장치를 새 OS에 맞게 다시 구성합니다.

### 첫번째 아래 이미지와 같이 
![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{3118D58A-7FC2-48D9-829E-474A5CBDE968}.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260819231014.png]]


https://www.raspberrypi.com/software/
![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{BC10422F-1A79-459A-910F-33CB8CA30D5D}.png]]

다운로드되면 exe파일이 나오는데 그것을 설치합니다.
![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{DD4C0F84-864A-4A69-B2B7-A1C6498D3D4F}.png]]

설치하고 실행을 하면 다음과 같은 창이 열립니다.
![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{20B1FC6B-9227-4944-BB5A-D262836526DA}.png]]

지금 화면에서는 **`Raspberry Pi 4`를 클릭**하면 됩니다.
```
Raspberry Pi OS (other) →`Raspberry Pi OS Lite (64-bit)
```
를 선택하세요.
![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{8AAB9DE3-FB6B-411F-AB1E-80B9547CC76C}.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{1A28CA16-2875-4CEF-BD76-DE2EED48DBAE}.png]]

호스트 네임은 지정한대로 넣어주세요. 그래야 충돌이 발생하지 않습니다.
```
학생 01 → rpi13-01
학생 02 → rpi13-02
학생 03 → rpi13-03
...
학생 30 → rpi13-30
```

사용자 이름과 비밀번호는 모두 같아도 됩니다. 

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{2D6148DC-B9E5-49CC-BE19-9603445125F4}.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{3422F6CC-B426-4ACA-8A97-0B3DB83D9BBE}.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{9177FE31-F1FB-4168-97D8-879D5C0E82B7}.png]]

WI-FI 비밀번호를 입력합니다.
![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/Pasted image 20260820112524.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{24C7D974-27CB-476D-888F-CC639898FC89}.png]]

![[{35018DEF-F2CB-4795-B946-E128984275DC}.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{5A1B047E-4C78-4648-B2F0-F8DD29C37CC6}.png]]

이제 SD카드 준비는 끝났고, 실제 Raspberry Pi를 처음 부팅해서 Wi-Fi → SSH 연결을 확인하는 단계입니다.

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{5DA43B21-1C62-4627-B51E-5996D0D66E9C}.png]]
지금 화면은 Raspberry Pi Imager가 microSD를 새 Raspberry Pi OS로 다시 기록한 뒤의 정상 상태입니다.


### 다음으로는
#### `1. SD카드를 안전하게 꺼냅니다`
윈도우 하단에 화살표를 눌러 안전하게 꺼내기를 합니다.
![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{71BA3CFB-42A0-4814-B486-61B51AA1B8D2}.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{0966FE50-B804-4060-816F-C0003569BEDD}.png]]


#### `2. microSD를 Raspberry Pi 아래쪽 슬롯에 넣습니다`
![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{51B386C5-2151-4ADA-A9E4-A39124BFB3EE}.png]]

#### `3. Raspberry Pi에 전원을 연결하세요`
![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{C17BDBF0-4E9C-4E1A-8A50-1EA2F16ED1CA}.png]]
빨간불과 초록불이 들어왔다면 전원이 들어오고 microSD를 읽으며 부팅을 시작한 상태입니다. 이제 Raspberry Pi를 더 만질 필요가 없습니다. 다음부터는 PC의 VS Code/WSL2에서 작업합니다.

우선 처음 부팅이므로 3~5분 정도 기다리세요. 그동안 Raspberry Pi가 우리가 SD카드에 넣어둔 설정을 읽어서 집 Wi-Fi에 접속하고 SSH 서비스를 시작합니다.

#### 전체 흐름정리
```
microSD 설치 완료
      ↓
Raspberry Pi에 삽입
      ↓
전원 ON              ← 지금 여기까지 성공
      ↓
Raspberry Pi OS 부팅
      ↓
집 Wi-Fi 연결
      ↓
SSH 서비스 시작
      ↓
PC에서 Raspberry Pi 찾기
      ↓
SSH 접속
```

#### `4. PC에서 VS Code를 여세요`
WSL2 환경으로 들어갑니다.
```
ssh edgepi@192.168.219.116
```

#### `5. Windows PowerShell에서 먼저 확인`
![[{3852BD4B-EC28-459F-B957-1F2AB73E54E1}.png]]

Windows 시작 메뉴에서 `PowerShell` 검색 → 실행 후: 권한으로 실행하지 않아도 됩니다. 
```powershell
ping rpi13-teacher
```

만약에 접속이 안될경우 사이트 http://192.168.219.32/login 직접 들어가서 설정합니다.
![[{DAA2DFE5-19DD-43CB-93F1-CEFD59CDA16E}.png]]

관리자 웹 접속 암호를 입력합니다.
![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{BDCDA7BD-F782-4AD6-B3A4-08D91BC78BBF}.png]]

![[교과 13 OnDevide EdgeAI_최종본/images-라즈베리젯슨/{3AB8A33C-322B-4E5F-9673-BA094678DCD8}.png]]

### 만약 Raspberry Pi에 SSH 연결이 되지 않을 경우
Raspberry Pi가 PC에서 바로 검색되지 않는다고 해서 운영체제를 다시 설치할 필요는 없습니다.

### 1. 먼저 3~5분 기다립니다

Raspberry Pi에 전원을 연결하면 처음 부팅하는 동안 다음 작업이 진행됩니다.
```
Raspberry Pi OS 부팅
       ↓
Wi-Fi 설정 적용
       ↓
공유기 연결
       ↓
IP 주소 할당
       ↓
SSH 서비스 시작
```
따라서 전원을 켠 직후 접속하지 말고 약 3~5분 정도 기다립니다.

### 2. Hostname으로 연결을 먼저 확인합니다

Windows PowerShell에서 다음 명령을 실행합니다.
```PowerShell
ping rpi13-teacher  # (자신이 설정했던 호스트네임입니다.)
```

다음과 같은 결과가 나오면 연결이 된겁니다.
```
2406:5900:102a:f173:da3a:ddff:fecd:b19f
```
Windows가 `rpi13-teacher`라는 이름을 찾아서 실제 Raspberry Pi의 **IPv6 주소**로 변환한 것입니다.

응답했다는 뜻
```
2406:5900:102a:f173:da3a:ddff:fecd:b19f의 응답: 시간=308ms
2406:5900:102a:f173:da3a:ddff:fecd:b19f의 응답: 시간=7ms
2406:5900:102a:f173:da3a:ddff:fecd:b19f의 응답: 시간=6ms
2406:5900:102a:f173:da3a:ddff:fecd:b19f의 응답: 시간=6ms
```
Raspberry Pi가 네 번 모두 응답했다는 뜻

PowerShell 터미널에서
```PowerShell
ssh edgepi@rpi13-teacher
```

Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
이렇게 입력하면

결과
```
Last login: Thu Aug 20 13:20:03 2026 from 192.168.219.114
```
이렇게 IPv4 확인을 할수 있습니다.

그다음 **VS Code의 WSL2**에서는 hostname 이름 해석이 안 됐으므로 이 IPv4로 들어갑니다.
```bash
ssh edgepi@192.168.219.116
```

그리고 패스워드를 입력합니다.
```
EdgeAI2026!
```


그러면 SSH 접속에 성공한 결과는 다음과 같습니다.
```bash
Linux rpi13-teacher 6.18.34+rpt-rpi-v8 #1 SMP PREEMPT Debian 1:6.18.34-1+rpt1 (2026-06-09) aarch64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Aug 20 15:23:57 2026 from 2406:5900:102a:f173:d52b:e21:aae3:6e37
```

### 확인용으로 다음 세개를 실행해봅니다.

현재 접속한 장비의 이름
```bash
hostname
```
결과
```
rpi13-teacher
```

Raspberry Pi에 할당된 네트워크 주소
```bash
hostname -I
```
결과
```
192.168.219.116 2406:5900:102a:f173:da3a:ddff:fecd:b19f 
```

Wi-Fi 연결 확인
```bash
nmcli device status
```
결과
```
DEVICE         TYPE      STATE                   CONNECTION              
wlan0          wifi      connected               netplan-wlan0-U+Net1410 
lo             loopback  connected (externally)  lo                      
p2p-dev-wlan0  wifi-p2p  disconnected            --                      
eth0           ethernet  unavailable             --    
```
→ `wlan0 connected`는 Wi-Fi가 정상 연결된 상태이고,  
→ `eth0 unavailable`은 LAN 케이블을 사용하지 않고 있다는 뜻입니다.