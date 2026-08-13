# Windows 전체 설정 가이드

이 문서는 Windows에 NUCLEO-F411RE를 USB로 연결해 STM32 펌웨어와 Raspberry
Pi 서버 코드를 함께 테스트하는 절차입니다. 명령은 PowerShell 기준입니다.

## 1. 필수 프로그램 설치 및 확인

- Git for Windows
- Python 3.10 이상
- STM32CubeIDE

Python 설치 시 `Add Python to PATH`를 선택합니다. PowerShell에서 확인합니다.

```powershell
git --version
py --version
```

## 2. 저장소 받기

```powershell
git clone https://github.com/PjongB/SAFE-KICK.git
cd SAFE-KICK
```

이미 받은 저장소를 갱신할 때는 다음 명령을 사용합니다.

```powershell
git pull origin main
```

## 3. STM32 프로젝트 열기 및 업로드

1. STM32CubeIDE를 실행합니다.
2. `File > Import`를 선택합니다.
3. `General > Existing Projects into Workspace`를 선택합니다.
4. `Select root directory`에 `SAFE-KICK\stm32`를 지정합니다.
5. 검색된 프로젝트를 선택하고 `Finish`를 누릅니다.
6. NUCLEO-F411RE를 USB로 연결합니다.
7. 프로젝트를 빌드한 후 `Run` 또는 `Debug`로 업로드합니다.
8. 보드가 재시작되면 발판을 비워 둔 채 `Tare...`, `READY` 순서를 기다립니다.

장치 관리자에 ST-LINK가 나타나지 않으면 STM32CubeIDE와 함께 제공되는
ST-LINK USB 드라이버를 확인합니다.

## 4. Python 테스트 환경 설정

```powershell
cd raspberry-pi
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements-test.txt
```

PowerShell이 가상환경 스크립트 실행을 차단하면 현재 터미널에서만 허용합니다.

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

## 5. COM 포트 확인

1. `장치 관리자 > 포트(COM 및 LPT)`를 엽니다.
2. `STMicroelectronics STLink Virtual COM Port (COMx)`를 찾습니다.

PowerShell에서도 확인할 수 있습니다.

```powershell
[System.IO.Ports.SerialPort]::GetPortNames()
```

예를 들어 `COM5`가 나오면 서버 설정에 `COM5`를 사용합니다. STM32CubeIDE
Serial Terminal, PuTTY, Tera Term 등에서 같은 포트를 열고 있다면 먼저
연결을 닫습니다.

## 6. 서버 실행

아래 `COM5`를 장치 관리자에서 확인한 실제 포트 번호로 변경합니다.

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

정상 실행 메시지:

```text
Application startup complete.
Uvicorn running on http://127.0.0.1:8000
```

## 7. Swagger 테스트

브라우저에서 [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)를
엽니다.

1. `GET /status`: `stm32_connected: true`, `is_locked: true` 확인
2. `POST /session/start`: 아래 데이터로 세션 생성
3. `POST /test/auth-complete`: MQ3 검사 시작
4. `GET /status`: `waiting_rider`, `unsafe: false` 확인
5. `POST /test/weight-check`: 한 명 탑승 측정 시작
6. 약 3초간 안정적으로 서 있기
7. `GET /status`: `is_locked: false`, `monitoring` 확인
8. 테스트 후 `POST /session/end` 실행

세션 생성 요청:

```json
{
  "user_id": 7,
  "kickboard_id": "KB-001",
  "face_vector": null
}
```

인증 및 무게 측정 요청:

```json
{
  "user_id": 7
}
```

## 8. 2인 탑승 및 잠금 테스트

`monitoring` 상태에서 총무게 `110kg` 이상을 4초 유지하면 부저가 울립니다.
그 상태를 추가로 10초 유지하면 `LOCK`이 실행됩니다. 실제 두 사람이
올라가기보다 킥보드를 고정하고 안전한 추나 물체를 사용하는 것을 권장합니다.

## 9. 종료 및 문제 해결

- 서버 종료: 서버 PowerShell에서 `Ctrl+C`
- COM 포트 오류: CubeIDE, PuTTY, Tera Term 등의 시리얼 연결 종료
- 포트 번호 오류: 장치 관리자에서 현재 COM 번호 재확인
- 가상환경 오류: `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass`
- 무게 오류: 보드 리셋 후 `Tare...` 동안 발판을 비워 두기
- WSL 사용 시: USB 장치가 자동 연결되지 않으므로 native PowerShell 실행을 권장
