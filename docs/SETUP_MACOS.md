# macOS 전체 설정 가이드

이 문서는 Mac에 NUCLEO-F411RE를 USB로 연결해 STM32 펌웨어와 Raspberry Pi
서버 코드를 함께 테스트하는 절차입니다.

## 1. 필수 프로그램 확인

- Git
- Python 3.10 이상
- STM32CubeIDE

터미널에서 확인합니다.

```bash
git --version
python3 --version
```

Git이 없다면 `xcode-select --install`로 Command Line Tools를 설치합니다.
Python은 python.org 설치 파일이나 Homebrew를 사용할 수 있습니다.

## 2. 저장소 받기

```bash
git clone https://github.com/PjongB/SAFE-KICK.git
cd SAFE-KICK
```

이미 받은 저장소를 갱신할 때는 다음 명령을 사용합니다.

```bash
git pull origin main
```

## 3. STM32 프로젝트 열기 및 업로드

1. STM32CubeIDE를 실행합니다.
2. `File > Import`를 선택합니다.
3. `General > Existing Projects into Workspace`를 선택합니다.
4. `Select root directory`에 `SAFE-KICK/stm32`를 지정합니다.
5. 검색된 프로젝트를 선택하고 `Finish`를 누릅니다.
6. NUCLEO-F411RE를 USB로 연결합니다.
7. 프로젝트를 빌드한 후 `Run` 또는 `Debug`로 업로드합니다.
8. 보드가 재시작되면 발판을 비워 둔 채 `Tare...`, `READY` 순서를 기다립니다.

STM32CubeIDE의 Serial Terminal을 사용해 확인했다면 Python 서버를 실행하기
전에 반드시 연결을 닫습니다. 한 시리얼 포트는 한 프로그램만 열 수 있습니다.

## 4. Python 테스트 환경 설정

```bash
cd raspberry-pi
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements-test.txt
```

터미널을 새로 열었을 때는 다시 프로젝트로 이동해 가상환경을 활성화합니다.

```bash
cd /path/to/SAFE-KICK/raspberry-pi
source .venv/bin/activate
```

## 5. USB UART 포트 확인

```bash
ls /dev/cu.usbmodem*
```

예시:

```text
/dev/cu.usbmodem1103
```

USB를 다시 연결하면 장치 번호가 달라질 수 있으므로 실행 전 확인합니다.
포트를 사용 중인 프로그램은 다음 명령으로 확인할 수 있습니다.

```bash
lsof /dev/cu.usbmodem1103
```

## 6. 서버 실행

아래 `SERIAL_PORT`를 실제로 확인한 장치명으로 변경합니다.

```bash
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

- 서버 종료: 서버 터미널에서 `Control+C`
- 포트 오류: CubeIDE Serial Terminal 등 다른 시리얼 프로그램 종료
- 연결 오류: 현재 `/dev/cu.usbmodem*` 번호와 `SERIAL_PORT` 재확인
- 무게 오류: 보드 리셋 후 `Tare...` 동안 발판을 비워 두기
