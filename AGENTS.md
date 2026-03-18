# Multi-Protocol-System

Simulator가 MQTT(로컬/컨테이너/클라우드 브로커)로 텔레메트리를 발행하고, Gateway가 수신 후 Kafka -> Consumer(PostgreSQL, SignalR) 흐름으로 처리하는 통합 시스템입니다.

- 서브모듈: [Multi-Protocol-Simulator/AGENTS.md](Multi-Protocol-Simulator/AGENTS.md), [Multi-Protocol-Gateway/AGENTS.md](Multi-Protocol-Gateway/AGENTS.md)

## Current Aggregation Structure

- 24시간 트렌드 차트:
  - 데이터 소스는 `sensor_agg_1hour` (1시간 버킷 집계)
  - 조회 API는 `/api/events/sensor-trends`
  - UI 자동 갱신 주기는 현재 15분
- 센서 읽기 카드:
  - 데이터 소스는 `sensor_agg_10min` (10분 버킷 집계)
  - 조회 API는 `/api/events/sensor-readings`
  - UI 자동 갱신 주기는 현재 5분
- 실시간 상세:
  - 센서 카드를 클릭하면 `RealtimeSensorChart` 모달이 열림
  - `SignalR` 허브(`/hubs/telemetry`)에 구독하여 실시간 값을 수신

## Repository Map

- `Multi-Protocol-Simulator`: 텔레메트리 생성 및 MQTT 발행 (Go CLI)
- `Multi-Protocol-Gateway`: MQTT 수신, Kafka 전달, Consumer 처리 (.NET API/UI)
- 루트 저장소는 서브모듈 기반 통합 테스트와 실행 가이드를 관리

## Working Rules

- 기능 변경은 반드시 해당 서브모듈 디렉터리에서 작업하고, 해당 서브모듈 `AGENTS.md` 지침을 우선 적용
- Simulator/Gateway 연동 동작 변경 시 두 서브모듈 지침을 교차 확인하고 통합(E2E) 검증
- 로컬 단위 검증보다 통합 검증 결과를 우선 근거로 사용

## Dev Environment Tips

- 테스트는 통합 환경 기준으로 진행
- Gateway: `cd Multi-Protocol-Gateway && docker compose up --build`
- Simulator: `.\simulator.exe run --config .\configs\docker.yaml`
- 상세 환경 가이드: [Multi-Protocol-Gateway/docs/DEVELOPMENT.md](Multi-Protocol-Gateway/docs/DEVELOPMENT.md)

## Testing Instructions

통합 환경에서만 검증합니다.

1. Gateway 기동: `cd Multi-Protocol-Gateway && docker compose up --build`
2. Simulator 발행: `.\simulator.exe run --config .\configs\docker.yaml`
3. 확인: Gateway를 통한 값 전달, SignalR 통신, Prometheus/Grafana 메트릭 갱신 정상 여부 확인

## PR Instructions

- 변경 시 해당 서브모듈(Simulator/Gateway) 지침에 따라 통합 E2E 확인 후 PR
- Conventional Commit 스타일 권장 (`feat:`, `fix:` 등)
