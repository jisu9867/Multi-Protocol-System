# Smart Factory Gateway & Simulator - 트러블슈팅 가이드

이 문서는 Smart Factory Integration Gateway와 Virtual Publisher (Protocol Simulator) 개발 및 통합 과정에서 발생한 주요 이슈와 해결 방법을 정리한 것입니다.

## 목차

1. [MQTT Adapter 연결 문제](#1-mqtt-adapter-연결-문제)
2. [Go Viper 설정 파일 로딩 문제](#2-go-viper-설정-파일-로딩-문제)
3. [MQTT 클라이언트 연결 상태 체크 문제](#3-mqtt-클라이언트-연결-상태-체크-문제)
4. [MQTT 토픽 패턴 불일치](#4-mqtt-토픽-패턴-불일치)
5. [PostgreSQL DateTime Kind 오류](#5-postgresql-datetime-kind-오류)
6. [Docker 네트워크 연결 문제](#6-docker-네트워크-연결-문제)
7. [Blazor UI Health Check 오류 표시 문제](#7-blazor-ui-health-check-오류-표시-문제)

---

## 1. MQTT Adapter 연결 문제

### 증상
- 시뮬레이터가 실행되지 않거나 MQTT 브로커에 연결 실패
- 에러 메시지: `failed to connect after 0 retries`

### 원인
MQTT Adapter의 재연결 로직에서 `ReconnectMaxRetries`가 0일 때 기본값 처리가 누락되어 재시도가 전혀 발생하지 않음.

### 해결 방법

**`simulator/internal/adapters/mqtt/mqtt.go`**

```go
func (a *MqttAdapter) Connect(ctx context.Context) error {
    // ...
    maxRetries := a.config.ReconnectMaxRetries
    if maxRetries == 0 {
        maxRetries = 10 // 기본값 설정
    }
    // ...
}
```

### 교훈
- 재시도 로직에서 0 값의 의미를 명확히 정의해야 함 (무제한 vs 재시도 없음)
- 설정값의 기본값 처리를 항상 고려해야 함

---

## 2. Go Viper 설정 파일 로딩 문제

### 증상
- YAML 설정 파일의 nested 필드가 Go struct에 제대로 매핑되지 않음
- `source_id`, `topic_template` 등이 빈 문자열로 로드됨

### 원인
Viper의 `Unmarshal`이 중첩된 구조체 필드를 제대로 파싱하지 못하는 경우가 있음. 특히 YAML의 `snake_case`와 Go의 `PascalCase` 간 매핑이 자동으로 되지 않을 수 있음.

### 해결 방법

**`simulator/internal/config/config.go`**

```go
func LoadConfig(configPath string) (*Config, error) {
    v := viper.New()
    v.SetConfigFile(configPath)
    v.SetConfigType("yaml") // 명시적으로 타입 지정
    
    if err := v.ReadInConfig(); err != nil {
        return nil, fmt.Errorf("failed to read config: %w", err)
    }
    
    var config Config
    if err := v.Unmarshal(&config); err != nil {
        return nil, fmt.Errorf("failed to unmarshal config: %w", err)
    }
    
    // Nested 필드 명시적 로딩
    if v.IsSet("generator.source_id") {
        config.Generator.SourceID = v.GetString("generator.source_id")
    }
    if v.IsSet("mqtt.topic_template") {
        config.MQTT.TopicTemplate = v.GetString("mqtt.topic_template")
    }
    
    return &config, nil
}
```

### 교훈
- Viper의 `Unmarshal`은 대부분의 경우 잘 작동하지만, nested 필드가 누락될 수 있음
- 중요한 설정값은 명시적으로 `GetString`, `UnmarshalKey` 등을 사용하여 검증하는 것이 안전함
- YAML 태그(`yaml:"source_id"`)를 struct 필드에 명시적으로 추가하는 것도 도움이 됨

---

## 3. MQTT 클라이언트 연결 상태 체크 문제

### 증상
- MQTT 클라이언트가 실제로는 연결되어 있지만 `IsConnected()`가 `false`를 반환
- 메시지 발행 시 "MQTT client not connected" 에러 발생

### 원인
MQTT 클라이언트의 `IsConnected()` 상태가 실제 연결 상태를 정확히 반영하지 못하는 경우가 있음. 특히 연결 직후나 일시적인 네트워크 지연 시 발생할 수 있음.

### 해결 방법

**`simulator/internal/adapters/mqtt/mqtt.go`**

```go
func (a *MqttAdapter) Publish(ctx context.Context, msg core.TelemetryMessage) error {
    if !a.client.IsConnected() {
        // 단일 재연결 시도
        if err := a.Connect(ctx); err != nil {
            return fmt.Errorf("client not connected and reconnect failed: %w", err)
        }
    }
    // ...
}
```

또는 Health Check에서:

**`gateway/src/Gateway.Adapters/MqttAdapter/MqttAdapter.cs`**

```csharp
// 단순히 IsConnected만 체크하지 않고, 실제 메시지 수신 여부도 고려
var isActuallyConnected = _status == AdapterStatus.Running && 
    (isConnected || _messageCount > 0 || (DateTime.UtcNow - _lastMessageTime).TotalSeconds < 60);
```

### 교훈
- `IsConnected()` 같은 상태 체크 메서드는 참고용으로만 사용하고, 실제 동작 여부는 다른 지표(메시지 수신, 최근 활동 시간 등)로 판단하는 것이 더 안정적
- 네트워크 연결은 일시적으로 끊길 수 있으므로, 재연결 로직을 항상 포함해야 함

---

## 4. MQTT 토픽 패턴 불일치

### 증상
- 시뮬레이터가 메시지를 발행하지만 Gateway가 수신하지 못함
- Gateway 로그에 메시지 수신 기록이 없음

### 원인
시뮬레이터의 토픽 구조와 Gateway의 구독 패턴이 일치하지 않음:
- **시뮬레이터**: `factory/{line}/{source_id}/telemetry` (예: `factory/line-1/sim-001/telemetry`)
- **Gateway**: `factory/+/telemetry` (2단계 wildcard만 매칭)

### 해결 방법

**`gateway/src/Gateway.Api/appsettings.json`**

```json
{
  "Adapters": {
    "Mqtt": {
      "Topic": "factory/+/+/telemetry"  // 3단계 wildcard로 변경
    }
  }
}
```

### 교훈
- MQTT 토픽 구조를 설계할 때는 wildcard(`+`, `#`)의 동작을 정확히 이해해야 함
  - `+`: 단일 레벨 매칭 (예: `factory/+/telemetry`는 `factory/line-1/telemetry`만 매칭)
  - `#`: 다중 레벨 매칭 (예: `factory/#`는 `factory/line-1/sim-001/telemetry` 매칭)
- 토픽 구조는 프로젝트 초기에 명확히 정의하고 문서화해야 함

---

## 5. PostgreSQL DateTime Kind 오류

### 증상
- Gateway가 PostgreSQL에 데이터를 저장할 때 다음 에러 발생:
  ```
  System.ArgumentException: Cannot write DateTime with Kind=Unspecified 
  to PostgreSQL type 'timestamp with time zone', only UTC is supported.
  ```

### 원인
.NET의 `DateTime`은 `DateTimeKind`가 `Unspecified`일 수 있으며, PostgreSQL의 `timestamp with time zone` 타입은 UTC만 지원함. MQTT 메시지에서 파싱한 `DateTime`이 `Unspecified`로 설정되어 있었음.

### 해결 방법

**여러 지점에서 UTC 변환 보장:**

1. **MQTT Adapter에서 파싱 시** (`MqttAdapter.cs`)

```csharp
private async Task OnMessageReceivedAsync(MqttApplicationMessageReceivedEventArgs e)
{
    // ...
    if (json.TryGetProperty("ts", out var tsProp))
    {
        var timestamp = tsProp.GetDateTimeOffset();
        timestamp = timestamp.ToUniversalTime(); // UTC 변환
        // ...
    }
}
```

2. **Normalize Stage에서** (`NormalizeStage.cs`)

```csharp
private static TelemetryEvent Normalize(RawData rawData)
{
    return new TelemetryEvent
    {
        // ...
        Timestamp = new DateTimeOffset(rawData.Timestamp, TimeSpan.Zero).ToUniversalTime(),
        // ...
    };
}
```

3. **PostgreSQL Sink에서 저장 전** (`PostgreSqlSink.cs`)

```csharp
private static TelemetryEventEntity MapToEntity(TelemetryEvent telemetryEvent)
{
    DateTime utcTimestamp;
    if (telemetryEvent.Timestamp.Offset == TimeSpan.Zero)
    {
        utcTimestamp = telemetryEvent.Timestamp.DateTime;
    }
    else
    {
        utcTimestamp = telemetryEvent.Timestamp.ToUniversalTime().DateTime;
    }
    
    // DateTimeKind가 Unspecified일 수 있으므로 명시적으로 UTC로 설정
    if (utcTimestamp.Kind != DateTimeKind.Utc)
    {
        utcTimestamp = DateTime.SpecifyKind(utcTimestamp, DateTimeKind.Utc);
    }
    
    return new TelemetryEventEntity
    {
        Timestamp = utcTimestamp,
        // ...
    };
}
```

### 교훈
- **항상 `DateTimeOffset`을 사용하라**: `DateTime`보다 타임존 정보를 명시적으로 다룰 수 있음
- **UTC로 통일**: 데이터베이스 저장 시 항상 UTC로 변환하고 `DateTimeKind.Utc`를 명시적으로 설정
- **파이프라인의 각 단계에서 검증**: 데이터가 파이프라인을 통과할 때마다 타임존 정보가 올바른지 확인
- **Npgsql의 요구사항**: PostgreSQL의 `timestamp with time zone`은 UTC만 지원하므로, 애플리케이션 레벨에서 보장해야 함

---

## 6. Docker 네트워크 연결 문제

### 증상
- Gateway API 컨테이너가 Mosquitto 브로커에 연결하지 못함
- `localhost:1883`로 연결 시도하지만 실패

### 원인
Docker 컨테이너 내부에서 `localhost`는 컨테이너 자체를 가리키므로, 호스트 머신의 `localhost`나 다른 컨테이너에 접근할 수 없음. Gateway와 Mosquitto가 서로 다른 Docker Compose 파일로 실행되어 서로 다른 네트워크에 있었음.

### 해결 방법

**옵션 1: 같은 Docker Compose 파일에 통합** (`gateway/docker-compose.yml`)

```yaml
services:
  mosquitto:
    image: eclipse-mosquitto:2.0
    container_name: gateway-mosquitto
    ports:
      - "1883:1883"
    volumes:
      - ./docker/mosquitto.conf:/mosquitto/config/mosquitto.conf

  api:
    # ...
    environment:
      - Adapters__Mqtt__Server=mosquitto  # localhost 대신 서비스 이름 사용
    depends_on:
      mosquitto:
        condition: service_started
```

**옵션 2: 호스트 네트워크 접근** (Windows/Mac)

```yaml
services:
  api:
    # ...
    environment:
      - Adapters__Mqtt__Server=host.docker.internal  # 호스트 머신 접근
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

### 교훈
- **Docker 네트워크 이해**: 컨테이너 간 통신은 서비스 이름을 사용해야 함
- **네트워크 격리**: 같은 Docker Compose 파일의 서비스는 기본적으로 같은 네트워크에 있음
- **호스트 접근**: Windows/Mac에서는 `host.docker.internal`을 사용하여 호스트 머신에 접근 가능
- **의존성 관리**: `depends_on`을 사용하여 서비스 시작 순서를 보장

---

## 7. Blazor UI Health Check 오류 표시 문제

### 증상
- Gateway가 실제로는 정상 작동하고 메시지를 수신하고 있지만, Blazor UI에 "MQTT client is not connected" 에러 메시지가 계속 표시됨

### 원인
Health Check 로직이 MQTT 클라이언트의 `IsConnected()` 속성만 체크하여, 실제 연결 상태나 메시지 수신 여부를 고려하지 않았음. 네트워크 지연이나 일시적인 상태 불일치로 인해 `IsConnected()`가 `false`를 반환할 수 있음.

### 해결 방법

**`gateway/src/Gateway.Adapters/MqttAdapter/MqttAdapter.cs`**

```csharp
public Task<AdapterHealth> GetHealthAsync(CancellationToken cancellationToken = default)
{
    lock (_metricsLock)
    {
        var isConnected = _mqttClient?.IsConnected ?? false;
        
        // 실제 연결 상태를 종합적으로 판단:
        // 1. 클라이언트가 연결되어 있거나
        // 2. 메시지를 받았거나
        // 3. 최근 60초 이내에 메시지를 받았으면 연결된 것으로 간주
        var isActuallyConnected = _status == AdapterStatus.Running && 
            (isConnected || _messageCount > 0 || 
             (DateTime.UtcNow - _lastMessageTime).TotalSeconds < 60);
        
        var health = new AdapterHealth
        {
            Status = _status,
            Metrics = new Dictionary<string, object>
            {
                ["is_connected"] = isActuallyConnected,
                ["client_connected"] = isConnected,
                ["last_message_seconds_ago"] = _lastMessageTime == DateTime.MinValue 
                    ? -1 
                    : (DateTime.UtcNow - _lastMessageTime).TotalSeconds
            }
        };

        // 에러 메시지는 실제 문제가 있을 때만 표시
        if (_status == AdapterStatus.Faulted)
        {
            health.ErrorMessage = "Adapter is in faulted state";
        }
        else if (_circuitState == CircuitState.Open)
        {
            health.ErrorMessage = $"Circuit breaker is open (failures: {_failureCount})";
        }
        else if (!isActuallyConnected && _status == AdapterStatus.Running)
        {
            // 메시지가 없고 120초 이상 지났을 때만 에러 표시
            if (_messageCount == 0 && 
                (DateTime.UtcNow - _lastMessageTime).TotalSeconds > 120)
            {
                health.ErrorMessage = "MQTT client is not connected";
            }
        }

        return Task.FromResult(health);
    }
}
```

### 교훈
- **Health Check는 실제 동작을 반영해야 함**: 단순한 상태 플래그보다는 실제 활동(메시지 수신, 최근 활동 시간 등)을 기반으로 판단
- **False Positive 방지**: 일시적인 상태 불일치로 인한 오류 표시를 방지하기 위해 여러 지표를 종합적으로 고려
- **사용자 경험**: UI에 표시되는 에러 메시지는 실제 문제가 있을 때만 표시되어야 함
- **Graceful Degradation**: 연결이 끊겼어도 최근에 메시지를 받았다면 일정 시간 동안은 정상으로 간주

---

## 일반적인 베스트 프랙티스

### 1. 에러 처리
- 모든 네트워크 작업에 재시도 로직 포함
- 타임아웃 설정 명시
- 에러 메시지는 구체적이고 디버깅에 유용한 정보 포함

### 2. 설정 관리
- 설정값의 기본값 명시
- 설정 파일 로딩 후 검증 로직 포함
- 중요한 설정값은 명시적으로 체크

### 3. 타임존 처리
- 항상 UTC로 저장
- `DateTimeOffset` 사용 권장
- 데이터베이스 저장 전 `DateTimeKind` 명시적 설정

### 4. Docker 네트워킹
- 컨테이너 간 통신은 서비스 이름 사용
- 호스트 접근이 필요한 경우 `host.docker.internal` 사용
- 서비스 의존성은 `depends_on`으로 관리

### 5. Health Check
- 실제 동작 여부를 반영하는 지표 사용
- False Positive 방지를 위한 여러 지표 종합 고려
- 사용자 경험을 고려한 에러 메시지 표시

---

## 참고 자료

- [MQTT Topic Wildcards](https://www.hivemq.com/blog/mqtt-essentials-part-5-mqtt-topics-best-practices/)
- [Npgsql DateTime Handling](https://www.npgsql.org/doc/types/datetime.html)
- [Docker Networking](https://docs.docker.com/network/)
- [Go Viper Configuration](https://github.com/spf13/viper)
- [MQTTnet Library](https://github.com/dotnet/MQTTnet)

