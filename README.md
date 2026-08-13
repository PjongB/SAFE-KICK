# SAFE-KICK

SAFE-KICK 전동 킥보드 안전 시스템의 Raspberry Pi 서버와 STM32 펌웨어를
한 저장소에서 관리합니다.

## Repository structure

```text
SAFE-KICK/
├── raspberry-pi/  # FastAPI, UART, MQ3/탑승 판단 및 테스트 API
└── stm32/         # STM32F411RE 펌웨어, 센서/릴레이/부저/UART 제어
```

## System flow

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

각 프로젝트의 설정과 실행 방법은 폴더별 README를 참고하세요.

- [Raspberry Pi](raspberry-pi/README.md)
- [STM32](stm32/README.md)

## Imported revisions

- Raspberry Pi: `safe-kick/safe-kick-raspi` commit `a8b7960`
- STM32: `safe-kick/safe-kick-stm32` commit `34ac5a7`

로컬 가상환경, SQLite 데이터베이스, 얼굴 임베딩, STM32 빌드 산출물은
저장소에 포함하지 않습니다.
