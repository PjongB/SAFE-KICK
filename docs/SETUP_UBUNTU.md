# Ubuntu 전체 설정 가이드

이 문서는 Ubuntu에 NUCLEO-F411RE를 USB로 연결해 STM32 펌웨어와 Raspberry
Pi 서버 코드를 함께 테스트하는 절차입니다.

## 1. 필수 패키지 설치

```bash
sudo apt update
sudo apt install git python3 python3-pip python3-venv
```

버전을 확인합니다.

```bash
git --version
python3 --version
```

STM32CubeIDE Linux 버전은 STMicroelectronics에서 내려받아 설치합니다.

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

## 4. Python 테스트 환경 설정

```bash
cd raspberry-pi
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements-test.txt
```

터미널을 새로 열었을 때는 다시 가상환경을 활성화합니다.

```bash
cd /path/to/SAFE-KICK/raspberry-pi
source .venv/bin/activate
```

## 5. USB UART 장치와 권한 확인

Nucleo는 일반적으로 `/dev/ttyACM0`으로 인식됩니다.

```bash
ls /dev/ttyACM*
dmesg | tail -n 30
```

시리얼 권한 오류를 예방하기 위해 사용자를 `dialout` 그룹에 추가합니다.

```bash
sudo usermod -aG dialout "$USER"
```

적용하려면 로그아웃 후 다시 로그인하거나 재부팅합니다. 다른 프로그램이
포트를 사용 중인지 확인할 때는 다음 명령을 사용합니다.

```bash
lsof /dev/ttyACM0
```

STM32CubeIDE Serial Terminal을 사용했다면 Python 서버 실행 전에 닫습니다.

## 6. 서버 실행

아래 장치명을 실제로 확인한 `/dev/ttyACM*` 값으로 변경합니다.

```bash
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

- 서버 종료: 서버 터미널에서 `Ctrl+C`
- 권한 오류: `groups`에 `dialout`이 있는지 확인하고 다시 로그인
- 포트 오류: 현재 `/dev/ttyACM*` 장치명과 `SERIAL_PORT` 재확인
- 포트 점유: CubeIDE Serial Terminal 등 다른 시리얼 프로그램 종료
- 무게 오류: 보드 리셋 후 `Tare...` 동안 발판을 비워 두기
- ST-LINK 접근 오류: STM32CubeIDE 설치 시 제공되는 udev rules 설치 확인
