# SAFE-KICK

SAFE-KICK 전동 킥보드 안전 시스템의 Raspberry Pi 서버와 STM32 펌웨어를 한 저장소에서 관리합니다.

## 운영체제별 전체 설정 가이드

사용 중인 운영체제에 맞는 문서를 선택하면 저장소 다운로드부터 STM32
펌웨어 업로드, Python 서버 실행, UART 연결, API 테스트까지 순서대로 진행할
수 있습니다.

- [macOS 전체 설정 가이드](docs/SETUP_MACOS.md)
- [Windows 전체 설정 가이드](docs/SETUP_WINDOWS.md)
- [Ubuntu 전체 설정 가이드](docs/SETUP_UBUNTU.md)
- [Confluence용 시스템 문서](docs/CONFLUENCE_SAFE_KICK.md)

## 시스템 구성

```text
SAFE-KICK/
├── raspberry-pi/  # FastAPI, UART, MQ3/탑승 판단 및 테스트 API
└── stm32/         # STM32F411RE 펌웨어, 센서/릴레이/부저/UART 제어
```

```text
STM32 연결 및 LOCK
-> 앱 인증 완료
-> MQ3 검사
-> 탑승 무게 검사
-> UNLOCK
-> 주행 중 2인 탑승 감시
-> 경고 부저
-> 지속 시 LOCK
```

## 1. 준비물

- Git
- Python 3.10 이상
- STM32CubeIDE
- NUCLEO-F411RE
- MQ-3, HX711 및 로드셀 4개, 릴레이, 부저
- Raspberry Pi 또는 USB로 STM32를 연결할 테스트 PC

## 2. 저장소 받기

```bash
git clone https://github.com/PjongB/SAFE-KICK.git
cd SAFE-KICK
```

기존 저장소를 갱신할 때는 다음 명령을 사용합니다.

```bash
git pull origin main
```

## 3. STM32 설정

1. STM32CubeIDE를 실행합니다.
2. `File > Import`를 선택합니다.
3. `General > Existing Projects into Workspace`를 선택합니다.
4. `Select root directory`에 내려받은 저장소의 `stm32` 폴더를 지정합니다.
5. 검색된 프로젝트를 선택하고 `Finish`를 누릅니다.
6. NUCLEO-F411RE를 USB로 연결합니다.
7. 프로젝트를 빌드한 후 `Run` 또는 `Debug`로 펌웨어를 업로드합니다.

| 장치 | 핀 |
|---|---|
| USART2 TX / RX | PA2 / PA3 |
| MQ-3 ADC | PA1 |
| BUZZER | PA6 |
| RELAY | PA7 |
| HX711 FL DT / SCK | PB0 / PB1 |
| HX711 FR DT / SCK | PB2 / PB10 |
| HX711 RL DT / SCK | PB12 / PB13 |
| HX711 RR DT / SCK | PB14 / PB15 |

USART2 설정은 `115200-8-N-1`, flow control 없음입니다. 보드 부팅 시 로드셀 영점을 잡으므로 `Tare...`가 출력되는 동안 발판을 비워 두고 `READY`가 나온 후 테스트합니다.

- [STM32 프로젝트 설명](stm32/README.md)
- [STM32 시리얼 테스트](stm32/README_SERIAL_TEST.md)

## 4. Raspberry Pi 서버 설정

```bash
cd raspberry-pi
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
```

백엔드와 얼굴 인식 없이 STM32 통합 테스트만 할 때는 경량 의존성을 설치합니다.

```bash
python3 -m pip install -r requirements-test.txt
```

얼굴 인증까지 사용할 때는 전체 의존성을 설치합니다.

```bash
python3 -m pip install -r requirements.txt
```

다음에 다시 실행할 때는 프로젝트로 이동해 가상환경만 활성화하면 됩니다.

```bash
cd /path/to/SAFE-KICK/raspberry-pi
source .venv/bin/activate
```

## 5. UART 연결

### Mac과 Nucleo USB 연결

```bash
ls /dev/cu.usbmodem*
```

예를 들어 `/dev/cu.usbmodem1103`이 나오면 이 값을 `SERIAL_PORT`에 사용합니다. STM32CubeIDE 시리얼 모니터 등 다른 프로그램이 포트를 열고 있으면 먼저 닫아야 합니다.

### Windows와 Nucleo USB 연결

1. Nucleo를 USB로 연결합니다.
2. `장치 관리자 > 포트(COM 및 LPT)`를 엽니다.
3. `STMicroelectronics STLink Virtual COM Port (COMx)`의 포트 번호를 확인합니다.

PowerShell에서도 현재 COM 포트 목록을 확인할 수 있습니다.

```powershell
[System.IO.Ports.SerialPort]::GetPortNames()
```

예를 들어 `COM5`가 나오면 `SERIAL_PORT=COM5`로 사용합니다. 장치 관리자에
포트가 보이지 않으면 STM32CubeIDE에 포함된 ST-LINK 드라이버를 설치하거나
STMicroelectronics의 ST-LINK USB 드라이버를 설치한 뒤 보드를 다시
연결합니다.

STM32CubeIDE Serial Terminal, PuTTY, Tera Term 등에서 같은 COM 포트를 열고
있으면 Python 서버가 포트를 열 수 없으므로 먼저 연결을 종료합니다.

### Ubuntu와 Nucleo USB 연결

Nucleo는 일반적으로 `/dev/ttyACM0`으로 인식됩니다.

```bash
ls /dev/ttyACM*
```

연결 직후 장치 이름을 확인하려면 다음 명령도 사용할 수 있습니다.

```bash
dmesg | tail -n 30
```

시리얼 포트 권한 오류가 발생하면 현재 사용자를 `dialout` 그룹에 추가합니다.

```bash
sudo usermod -aG dialout "$USER"
```

적용하려면 Ubuntu에서 로그아웃 후 다시 로그인하거나 재부팅해야 합니다.
예를 들어 `/dev/ttyACM0`이 확인되면 이 값을 `SERIAL_PORT`에 사용합니다.

다른 프로그램이 포트를 사용 중인지 확인할 때는 다음 명령을 사용합니다.

```bash
lsof /dev/ttyACM0
```

### Raspberry Pi GPIO UART 연결

| Raspberry Pi | STM32 |
|---|---|
| TX | PA3, USART2 RX |
| RX | PA2, USART2 TX |
| GND | GND |

TX와 RX는 교차 연결하고 두 보드의 GND를 반드시 공통으로 연결합니다. 3.3V UART 레벨을 사용하며 5V를 UART 핀에 직접 연결하지 않습니다.

Raspberry Pi에서 `sudo raspi-config`를 실행하고 `Interface Options > Serial Port`에서 로그인 셸은 비활성화하고 시리얼 포트 하드웨어는 활성화한 뒤 재부팅합니다. 일반적인 장치 경로는 `/dev/serial0`입니다.

## 6. 백엔드 없이 Mock 테스트

STM32를 연결하지 않고 API 흐름부터 확인할 수 있습니다.

```bash
cd raspberry-pi
source .venv/bin/activate

ENABLE_FACE_API=false \
ENABLE_TEST_API=true \
USE_UART_MOCK=true \
python3 -m uvicorn app.main:app --host 127.0.0.1 --port 8000
```

브라우저에서 [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)를 열어 Swagger API 화면을 사용합니다.

## 7. 실제 STM32 통합 테스트

### Mac에서 실행

`SERIAL_PORT`는 앞에서 확인한 실제 장치명으로 바꿉니다.

```bash
cd raspberry-pi
source .venv/bin/activate

ENABLE_FACE_API=false \
ENABLE_TEST_API=true \
USE_UART_MOCK=false \
SERIAL_PORT=/dev/cu.usbmodem1103 \
BAUD_RATE=115200 \
python3 -m uvicorn app.main:app \
  --host 127.0.0.1 \
  --port 8000 \
  --log-level debug
```

### Windows PowerShell에서 실행

처음 설정하는 경우 저장소의 `raspberry-pi` 폴더에서 가상환경과 테스트
의존성을 준비합니다.

```powershell
cd raspberry-pi
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements-test.txt
```

PowerShell이 스크립트 실행을 차단하면 현재 터미널에서만 허용한 뒤 다시
활성화합니다.

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

장치 관리자에서 확인한 COM 포트를 지정해 서버를 실행합니다.

```powershell
$env:ENABLE_FACE_API="false"
$env:ENABLE_TEST_API="true"
$env:USE_UART_MOCK="false"
$env:SERIAL_PORT="COM5"
$env:BAUD_RATE="115200"

python -m uvicorn app.main:app `
  --host 127.0.0.1 `
  --port 8000 `
  --log-level debug
```

### Ubuntu에서 실행

`SERIAL_PORT`는 앞에서 확인한 `/dev/ttyACM*` 장치명으로 바꿉니다.

```bash
cd raspberry-pi
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements-test.txt

ENABLE_FACE_API=false \
ENABLE_TEST_API=true \
USE_UART_MOCK=false \
SERIAL_PORT=/dev/ttyACM0 \
BAUD_RATE=115200 \
python3 -m uvicorn app.main:app \
  --host 127.0.0.1 \
  --port 8000 \
  --log-level debug
```

Ubuntu에서 `python3 -m venv`가 실패하면 다음 패키지를 설치한 뒤 다시
시도합니다.

```bash
sudo apt update
sudo apt install python3-venv
```

### Raspberry Pi에서 실행

```bash
cd raspberry-pi
source .venv/bin/activate

ENABLE_FACE_API=false \
ENABLE_TEST_API=true \
USE_UART_MOCK=false \
SERIAL_PORT=/dev/serial0 \
BAUD_RATE=115200 \
python3 -m uvicorn app.main:app \
  --host 0.0.0.0 \
  --port 8000 \
  --log-level debug
```

같은 네트워크의 다른 기기에서는 `http://<라즈베리파이-IP>:8000/docs`로 접속합니다.

## 8. API 테스트 순서

Swagger에서 아래 순서대로 실행합니다. 킥보드는 움직이지 않게 고정하고 가능하면 구동 바퀴를 지면에서 띄운 상태로 테스트합니다.

### 8-1. 연결 확인

`GET /status`를 실행합니다. `stm32_connected: true`, `is_locked: true`, `safety_state: locked`, `uart_mock: false`인지 확인합니다.

### 8-2. 세션 생성

`POST /session/start`에 입력합니다.

```json
{
  "user_id": 7,
  "kickboard_id": "KB-001",
  "face_vector": null
}
```

응답의 `session_id`는 잠금과 결과 조회에 사용하므로 기억합니다.

### 8-3. MQ3 검사 시작

`POST /test/auth-complete`에 입력합니다.

```json
{
  "user_id": 7
}
```

완료 후 `GET /status`에서 `safety_state: waiting_rider`와 `alcohol_result.unsafe: false`를 확인합니다. `alcohol_result`에는 baseline, 8개 측정값, peak delta도 함께 표시됩니다.

### 8-4. 주행 전 한 명 탑승 확인

한 명이 올라갈 준비를 한 뒤 `POST /test/weight-check`에 입력합니다.

```json
{
  "user_id": 7
}
```

발판에 올라가 약 3초 동안 최대한 안정적으로 서 있습니다. 최근 무게 3개가 `20~100kg`이고 최대·최소 차이가 `5kg` 이내이면 자동으로 `UNLOCK`합니다.

`GET /status`에서 `is_locked: false`, `status: unlocked`, `safety_state: monitoring`인지 확인합니다.

### 8-5. 주행 중 2인 탑승 판단

`monitoring` 상태에서 총무게가 `110kg` 이상으로 4초간 유지되면 부저가 울리고 `warning` 상태가 됩니다. 그 상태가 추가로 10초 유지되면 STM32에 `LOCK`을 보내 릴레이를 차단합니다.

실제 두 사람이 올라가기보다 킥보드를 고정한 상태에서 안전한 추나 물체로 시험하는 것을 권장합니다. 무게가 `110kg` 아래로 내려가면 경고와 타이머가 초기화됩니다.

### 8-6. 테스트 종료

`POST /session/end`를 실행합니다. 입력값은 없습니다. 종료 과정에서 `LOCK`을 전송하고 세션 결과를 저장합니다.

`GET /session/summary`에 앞에서 받은 `session_id`를 넣으면 시작·종료 시각과 경고 횟수를 확인할 수 있습니다.

## 9. 테스트 명령 요약

```text
GET  /status
POST /session/start
POST /test/auth-complete
GET  /status
POST /test/weight-check
GET  /status
POST /session/end
GET  /session/summary
```

`GET /session/stream`은 앱에서 상태를 실시간으로 받을 때 사용하는 SSE API입니다. 센서 판단을 작동시키기 위해 직접 실행할 필요는 없습니다. `POST /unlock`은 안전 절차 우회를 막기 위해 기본적으로 비활성화되어 있습니다.

## 10. 문제 해결

### 시리얼 포트를 열 수 없는 경우

- STM32CubeIDE 시리얼 모니터나 다른 터미널 프로그램을 닫습니다.
- `SERIAL_PORT`가 현재 장치명과 같은지 다시 확인합니다.
- USB 케이블을 다시 연결하면 macOS 장치 번호가 바뀔 수 있습니다.
- Windows에서는 장치 관리자의 COM 번호와 `SERIAL_PORT`가 같은지 확인합니다.
- Ubuntu에서는 `dialout` 그룹 적용을 위해 다시 로그인했는지 확인합니다.

### `stm32_connected`가 `false`인 경우

- STM32 펌웨어가 실행 중인지 확인합니다.
- Baud rate가 양쪽 모두 `115200`인지 확인합니다.
- GPIO UART 연결에서는 TX/RX 교차 연결과 공통 GND를 확인합니다.

### 무게가 이상한 경우

- 보드 부팅 중 `Tare...` 단계에서 발판을 비워 둡니다.
- 로드셀 설치나 하중 위치가 바뀌었다면 scale 값을 다시 보정합니다.
- 보정 방법은 [STM32 시리얼 테스트 문서](stm32/README_SERIAL_TEST.md)를 참고합니다.

## 세부 문서

- [Raspberry Pi](raspberry-pi/README.md)
- [STM32](stm32/README.md)
- [STM32 UART 테스트](stm32/README_SERIAL_TEST.md)

## 가져온 기준 커밋

- Raspberry Pi: `safe-kick/safe-kick-raspi` commit `a8b7960`
- STM32: `safe-kick/safe-kick-stm32` commit `34ac5a7`

로컬 가상환경, SQLite 데이터베이스, 얼굴 임베딩, STM32 빌드 산출물은 저장소에 포함하지 않습니다.
