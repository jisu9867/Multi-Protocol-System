# Edge Gateway 아키텍처 분석 및 프로토콜 확장 제안

> **목적**: Edge Gateway에서 값을 받아오는 서버 관점에서 프로젝트 구조 평가 및 필요한 프로토콜 제안

---

## 1. 현재 프로젝트 구조 평가

### 1.1 아키텍처 구조

현재 프로젝트는 **Clean Architecture**를 잘 따르고 있습니다:

```
Gateway/
├── Gateway.Core/          ✅ 도메인 모델, 인터페이스 (프로토콜 독립)
├── Gateway.Infrastructure/ ✅ 인프라 구현 (PostgreSQL, File)
├── Gateway.Adapters/      ✅ 프로토콜별 어댑터 (확장 가능)
├── Gateway.Api/           ✅ 애플리케이션 진입점
└── Gateway.Ui/            ✅ 모니터링 UI
```

**장점**:
- ✅ **프로토콜 독립적인 Core**: 프로토콜이 변경되어도 Core 로직 영향 없음
- ✅ **Pipeline 패턴**: Ingest → Normalize → Route → Sink의 명확한 데이터 흐름
- ✅ **어댑터 인터페이스**: `IAdapter` 인터페이스로 확장 용이
- ✅ **의존성 역전**: Core가 Infrastructure에 의존하지 않음
- ✅ **Observability**: Health Check, Metrics, Logging 지원

**개선 필요 사항**:
- ⚠️ **어댑터 인스턴스 관리**: 현재는 설정 기반 단일 인스턴스만 지원
- ⚠️ **동적 어댑터 등록**: 런타임에 어댑터 추가/제거 불가
- ⚠️ **멀티 테넌트 지원**: 여러 Edge Gateway에서 데이터 구분 필요

---

## 2. Edge Gateway 환경에서 필요한 프로토콜

### 2.1 필수 프로토콜 (우선순위 높음)

#### 1. **OPC-UA (OPC Unified Architecture)** ⭐⭐⭐
- **용도**: 산업용 자동화 표준 프로토콜, PLC/SCADA 시스템 통신
- **특징**: 
  - 보안 내장 (암호화, 인증)
  - 정보 모델링 (노드 구조)
  - Subscriptions (변경 알림)
- **Edge Gateway 시나리오**: 
  - PLC에서 실시간 데이터 수집
  - SCADA 시스템과 통합
- **구현 난이도**: 높음 (복잡한 정보 모델)
- **라이브러리**: `OPCFoundation.NetStandard.Opc.Ua` (.NET)

#### 2. **Modbus TCP/RTU** ⭐⭐⭐
- **용도**: 산업용 프로토콜, PLC 통신의 사실상 표준
- **특징**:
  - 단순한 프로토콜 구조
  - Master/Slave 구조
  - Holding Registers, Input Registers, Coils 읽기/쓰기
- **Edge Gateway 시나리오**:
  - PLC에서 Modbus로 데이터 읽기
  - RTU (Serial) 또는 TCP (Ethernet) 지원
- **구현 난이도**: 중간
- **라이브러리**: `NModbus4` (.NET)

#### 3. **HTTP REST API** ⭐⭐
- **용도**: Edge Gateway가 직접 HTTP POST로 데이터 전송
- **특징**:
  - 표준 웹 프로토콜
  - 인증 (JWT, API Key)
  - 배치 전송 가능
- **Edge Gateway 시나리오**:
  - Edge Gateway 내부 로직에서 수집한 데이터를 REST API로 전송
  - 클라우드 동기화
- **구현 난이도**: 낮음 (이미 ASP.NET Core 기반)
- **라이브러리**: 내장 (`Microsoft.AspNetCore.Mvc`)

#### 4. **gRPC** ⭐⭐
- **용도**: 고성능 RPC, 마이크로서비스 간 통신
- **특징**:
  - HTTP/2 기반
  - Protocol Buffers (이진 직렬화)
  - 양방향 스트리밍
- **Edge Gateway 시나리오**:
  - Edge Gateway에서 고속 스트리밍 데이터 전송
  - 실시간 대량 데이터 처리
- **구현 난이도**: 중간
- **라이브러리**: `Grpc.AspNetCore` (.NET)

---

### 2.2 권장 프로토콜 (우선순위 중간)

#### 5. **WebSocket** ⭐⭐
- **용도**: 실시간 양방향 통신
- **특징**:
  - 지속 연결
  - 낮은 지연 시간
  - 서버 → 클라이언트 푸시 가능
- **Edge Gateway 시나리오**:
  - 실시간 모니터링 데이터 전송
  - 명령 제어 (Gateway → Edge)
- **구현 난이도**: 낮음
- **라이브러리**: 내장 (`Microsoft.AspNetCore.WebSockets`)

#### 6. **Serial (RS232/RS485)** ⭐
- **용도**: 시리얼 통신, 레거시 장비
- **특징**:
  - 물리적 시리얼 포트 통신
  - Modbus RTU 등 프로토콜의 전송 계층
- **Edge Gateway 시나리오**:
  - 시리얼 장비 직접 연결
  - RS485 버스 (다중 장비)
- **구현 난이도**: 중간
- **라이브러리**: `System.IO.Ports` (.NET)

#### 7. **CoAP (Constrained Application Protocol)** ⭐
- **용도**: IoT 장치용 경량 프로토콜 (MQTT 대안)
- **특징**:
  - UDP 기반
  - HTTP와 유사한 RESTful API
  - 리소스 제약 장치용
- **Edge Gateway 시나리오**:
  - 리소스가 제한된 Edge 장치에서 데이터 수집
- **구현 난이도**: 중간
- **라이브러리**: `CoAP.NET`

---

### 2.3 선택적 프로토콜 (특수 용도)

#### 8. **CAN Bus** 
- **용도**: 자동차/산업용 버스 프로토콜
- **시나리오**: 자동차, 로봇 제어 시스템
- **구현 난이도**: 높음 (하드웨어 의존)

#### 9. **Ethernet/IP**
- **용도**: Rockwell Automation 프로토콜
- **시나리오**: Rockwell PLC 통합
- **구현 난이도**: 높음

#### 10. **MQTT-SN (MQTT for Sensor Networks)**
- **용도**: MQTT의 UDP 버전
- **시나리오**: 무선 센서 네트워크
- **구현 난이도**: 중간

---

## 3. 프로토콜별 구현 우선순위 및 난이도

| 프로토콜 | 우선순위 | 구현 난이도 | 사용 빈도 | 구현 시간 (추정) |
|---------|---------|------------|----------|----------------|
| OPC-UA | 높음 | 높음 | 매우 높음 | 2-3주 |
| Modbus TCP | 높음 | 중간 | 매우 높음 | 1주 |
| HTTP REST | 높음 | 낮음 | 높음 | 2-3일 |
| gRPC | 중간 | 중간 | 높음 | 1주 |
| WebSocket | 중간 | 낮음 | 중간 | 3-5일 |
| Serial | 낮음 | 중간 | 중간 | 1주 |
| CoAP | 낮음 | 중간 | 낮음 | 1주 |

---

## 4. 프로토콜별 구현 구조 제안

### 4.1 어댑터 디렉토리 구조

```
Gateway.Adapters/
├── FakeAdapter/           (테스트용)
├── MqttAdapter/           ✅ 구현 완료
├── OpcUaAdapter/          📋 추천 구현
├── ModbusAdapter/         📋 추천 구현
├── HttpRestAdapter/       📋 추천 구현
├── GrpcAdapter/           📋 권장
├── WebSocketAdapter/      📋 권장
└── SerialAdapter/         📋 선택적
```

### 4.2 설정 구조 예시

```json
{
  "Adapters": {
    "Mqtt": {
      "Enabled": true,
      "Server": "localhost",
      "Port": 1883,
      "Topic": "factory/+/+/telemetry"
    },
    "OpcUa": {
      "Enabled": true,
      "EndpointUrl": "opc.tcp://localhost:4840",
      "SecurityPolicy": "Basic256Sha256",
      "NodeIds": [
        "ns=2;s=Temperature",
        "ns=2;s=Pressure"
      ],
      "SubscriptionInterval": 1000
    },
    "Modbus": {
      "Enabled": true,
      "Type": "Tcp",
      "Host": "192.168.1.100",
      "Port": 502,
      "UnitId": 1,
      "Registers": [
        {
          "Tag": "temp",
          "Address": 40001,
          "Type": "HoldingRegister",
          "DataType": "Float32"
        }
      ],
      "PollInterval": 1000
    },
    "HttpRest": {
      "Enabled": true,
      "BasePath": "/api/v1/telemetry",
      "Authentication": {
        "Type": "ApiKey",
        "Header": "X-API-Key"
      }
    },
    "Grpc": {
      "Enabled": true,
      "Port": 50051,
      "ServiceName": "TelemetryService"
    }
  }
}
```

---

## 5. 구현 권장 사항

### 5.1 단계별 구현 전략

#### Phase 1: 핵심 프로토콜 (1-2개월)
1. **Modbus TCP Adapter** (1주)
   - 가장 널리 사용되는 프로토콜
   - 구현 난이도 적당
   - 즉시 활용 가능

2. **HTTP REST Adapter** (2-3일)
   - 이미 ASP.NET Core 기반이므로 구현 용이
   - 범용성 높음

#### Phase 2: 산업 표준 (2-3개월)
3. **OPC-UA Adapter** (2-3주)
   - 산업 표준이므로 필수
   - 복잡하지만 장기적으로 중요

4. **gRPC Adapter** (1주)
   - 고성능 요구사항 대응

#### Phase 3: 확장 (선택적)
5. WebSocket, Serial, CoAP 등

---

### 5.2 공통 구현 패턴

모든 어댑터는 다음 패턴을 따라야 합니다:

```csharp
public class XxxAdapter : IAdapter, IAsyncDisposable
{
    // 1. 필수 인터페이스 구현
    public string Id { get; }
    public AdapterStatus Status { get; }
    
    // 2. 연결 관리
    public Task StartAsync(CancellationToken cancellationToken);
    public Task StopAsync(CancellationToken cancellationToken);
    
    // 3. Health Check
    public Task<AdapterHealth> GetHealthAsync(CancellationToken cancellationToken);
    
    // 4. 데이터 핸들러
    private readonly IAdapterDataHandler? _dataHandler;
    
    // 5. 메트릭 수집
    private long _messageCount;
    private long _reconnectCount;
    
    // 6. Circuit Breaker (선택적)
    private CircuitState _circuitState;
    
    // 7. Graceful Shutdown
    private CancellationTokenSource? _cancellationTokenSource;
}
```

---

## 6. 아키텍처 개선 제안

### 6.1 다중 어댑터 인스턴스 지원

현재는 설정 기반 단일 인스턴스만 지원합니다. 다음 개선이 필요합니다:

```csharp
// 현재: 단일 인스턴스
if (adapterOptions.Mqtt.Enabled)
{
    builder.Services.AddSingleton<IAdapter>(...);
}

// 개선: 다중 인스턴스 지원
if (adapterOptions.Mqtt.Instances != null)
{
    foreach (var mqttInstance in adapterOptions.Mqtt.Instances)
    {
        builder.Services.AddSingleton<IAdapter>(sp => 
            new MqttAdapter(mqttInstance.Id, mqttInstance.Options, ...));
    }
}
```

### 6.2 동적 어댑터 관리

런타임에 어댑터 추가/제거/재시작 기능:

```csharp
public interface IAdapterManager
{
    Task<IAdapter> AddAdapterAsync(string adapterType, string instanceId, object config);
    Task RemoveAdapterAsync(string instanceId);
    Task RestartAdapterAsync(string instanceId);
    Task<IEnumerable<IAdapter>> GetAdaptersAsync();
}
```

### 6.3 Edge Gateway 식별

여러 Edge Gateway에서 데이터를 구분하기 위한 메타데이터:

```csharp
public class RawData
{
    public string AdapterId { get; set; }
    public string SourceId { get; set; }
    public string? EdgeGatewayId { get; set; }  // 추가
    public string? EdgeGatewayLocation { get; set; }  // 추가
    public DateTime Timestamp { get; set; }
    // ...
}
```

---

## 7. 결론 및 추천

### 현재 구조 평가: ⭐⭐⭐⭐ (4/5)

**강점**:
- Clean Architecture로 확장 용이
- Pipeline 패턴으로 데이터 흐름 명확
- 인터페이스 기반 설계로 프로토콜 추가 용이

**개선 필요**:
- 다중 어댑터 인스턴스 지원
- 동적 어댑터 관리
- Edge Gateway 식별 메커니즘

### 필수 구현 프로토콜 (우선순위 순)

1. ✅ **MQTT** - 이미 구현됨
2. 🔥 **Modbus TCP** - 가장 널리 사용, 구현 용이
3. 🔥 **HTTP REST** - 범용성 높음, 구현 용이
4. ⭐ **OPC-UA** - 산업 표준, 필수
5. ⭐ **gRPC** - 고성능 요구사항

### 구현 로드맵

```
Month 1-2: Modbus TCP + HTTP REST
Month 3-4: OPC-UA
Month 5:   gRPC
Month 6+:  WebSocket, Serial 등 선택적
```

---

**작성일**: 2026-01-08  
**작성자**: Platform Engineer  
**목적**: Edge Gateway 서버 아키텍처 평가 및 확장 전략


┌─────────────────────────────────────────────────────────┐
│                   클라우드 (Azure)                       │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Gateway Server (현재 프로젝트)                     │  │
│  │  - MQTT (Subscribe) ✅                            │  │
│  │  - HTTP REST (API Endpoint) ✅                    │  │
│  │  - gRPC ✅                                        │  │
│  │  - WebSocket ✅                                   │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          ↑ 인터넷
                          │
        ┌─────────────────┴─────────────────┐
        │                                   │
┌───────▼────────┐                ┌─────────▼───────┐
│  Edge Gateway  │                │  Edge Gateway   │
│   (현장 #1)     │                │   (현장 #2)     │
│                │                │                 │
│  MQTT Client   │                │  HTTP REST      │
│  (Publish)     │                │  Client         │
│                │                │                 │
│  ┌──────────┐  │                │  ┌──────────┐   │
│  │ Serial   │◄─┤                │  │ Modbus   │ ◄─┤
│  │ Adapter  │  │                │  │ Adapter  │   │
│  └────┬─────┘  │                │  └────┬─────┘   │
│       │        │                │       │         │
│  ┌────▼────┐   │                │  ┌────▼────┐    │
│  │  PLC    │   │                │  │  PLC    │    │
│  └─────────┘   │                │  └─────────┘    │
└────────────────┘                └─────────────────┘