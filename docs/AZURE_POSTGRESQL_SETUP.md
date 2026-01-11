# Azure PostgreSQL Flexible Server 설정 가이드 (최저가 기준)

> **대상**: Azure 12개월 무료 계정 사용자  
> **목적**: Gateway 애플리케이션을 Azure App Service에 배포하고 PostgreSQL Flexible Server (B1ms) 사용  
> **원칙**: 비용 최소화, 간단한 설정, 바로 동작하는 구성

---

## 📋 목차

1. [Azure PostgreSQL 생성 가이드](#1-azure-postgresql-생성-가이드)
2. [네트워크 / 보안 설정](#2-네트워크--보안-설정)
3. [.NET 애플리케이션 설정 변경](#3-net-애플리케이션-설정-변경)
4. [Azure App Service 연동 체크리스트](#4-azure-app-service-연동-체크리스트)
5. [비용 최소화 운영 팁](#5-비용-최소화-운영-팁)
6. [최종 요약](#6-최종-요약)

---

## 1. Azure PostgreSQL 생성 가이드

### 1.1 Azure Portal 접속 및 PostgreSQL Flexible Server 생성

1. **Azure Portal** (https://portal.azure.com) 접속
2. 상단 검색창에서 "PostgreSQL" 입력 → **"Azure Database for PostgreSQL flexible servers"** 선택
3. **"+ Create"** 클릭

### 1.2 기본 설정 (Basics) 탭

#### **Subscription**
- ✅ 무료 계정의 구독 선택

#### **Resource Group**
- ✅ 기존 Resource Group 사용 (없으면 새로 생성, 이름: `rg-gateway-dev` 등)

#### **Server name**
- ✅ 유니크한 이름 입력 (예: `gateway-postgres-dev-{고유번호}`)
- ⚠️ 전역적으로 유일해야 하므로 숫자나 랜덤 문자열 추가 권장
- ❌ 소문자, 숫자, 하이픈(-)만 사용 가능

#### **Region**
- ✅ **App Service와 동일한 Region 선택 (중요!)**
  - 예: `Korea Central (서울)`
  - 이유: 같은 Region이면 데이터 전송 비용 없음 + 지연 시간 최소화
- ❌ 다른 Region 선택 시 추가 네트워크 비용 발생

#### **PostgreSQL version**
- ✅ **15** (기본값, 최신 안정 버전)

#### **Workload type**
- ✅ **Development** 선택 (비용 최적화됨)

#### **Compute + Storage 탭**

##### **Compute tier**
- ✅ **Burstable** 선택
- ❌ General Purpose / Memory Optimized 선택 금지 (비용 10배 이상 증가)

##### **Compute size**
- ✅ **Standard_B1ms** 선택
  - **1 vCore / 2GB RAM**
  - ✅ 최저가 옵션
  - ❌ Standard_B1s (0.5 vCore)는 성능 부족 가능성 높음

##### **Storage**
- ✅ **32 GB** (최소값)
  - Burstable B1ms는 최소 32GB부터 시작
  - 개발/PoC에는 충분
  - ❌ 100GB 이상 설정 금지 (비용 증가)

##### **Storage auto-grow**
- ✅ **Enable auto-grow** 체크 해제 (비용 폭탄 방지)
  - 또는 **32GB까지 자동 증가**로 제한 설정
  - ❌ 무제한 자동 증가 금지

#### **Backup 탭**

##### **Backup redundancy**
- ✅ **Locally redundant (LRS)** 선택
  - 가장 저렴한 옵션
  - ❌ Geo-redundant (GRS) 선택 금지 (2배 이상 비용)

##### **Backup retention period**
- ✅ **7 days** (최소값)
  - 개발/PoC에는 충분
  - ❌ 35일 선택 금지 (비용 증가)

##### **Backup scheduling**
- ✅ 기본값 사용 (시간대만 필요시 조정)

### 1.3 네트워크 탭 (다음 섹션 참조)

### 1.4 Security 탭

#### **Authentication method**
- ✅ **PostgreSQL authentication** 선택 (기본값)
  - Microsoft Entra ID는 추가 설정 복잡, 개발에는 불필요

#### **Admin username**
- ✅ `gatewayadmin` 또는 원하는 이름 입력
- ⚠️ 기억하기 쉬운 이름 권장 (나중에 변경 불가)

#### **Password**
- ✅ 강한 비밀번호 입력 (대소문자, 숫자, 특수문자 포함)
- ⚠️ **반드시 비밀번호 저장해둘 것!** (복구 불가능)
- 예: `Gateway@2024!Dev`

### 1.5 Review + Create

1. **Review + Create** 클릭
2. 검증 통과 후 **Create** 클릭
3. 배포 완료까지 약 **5-10분** 소요
4. 배포 완료 후 **"Go to resource"** 클릭

---

## 2. 네트워크 / 보안 설정

### 2.1 Public Access 설정

1. 생성된 PostgreSQL 서버 리소스에서 좌측 메뉴 **"Networking"** 클릭
2. **"Public access"** 탭 선택

#### **Firewall rules**

##### ✅ 기본 규칙: Azure 서비스 접근 허용
- **"Allow public access from Azure services and resources within Azure"** 체크박스 ✅
  - App Service에서 DB 접근 가능하게 함
  - 보안 위험 최소 (Azure 내부 트래픽만)

##### ✅ 개발자 PC 접근 허용 (선택)
- **"+ Add current client IP address"** 클릭
  - 현재 IP 주소 자동 추가
  - 로컬 개발/디버깅용
  - ⚠️ IP 변경 시 재추가 필요

##### ✅ 특정 IP 범위 추가 (필요시)
- **"+ Add firewall rule"** 클릭
  - Rule name: `dev-pc` (예)
  - Start IP address: `1.2.3.4` (예)
  - End IP address: `1.2.3.4` (동일 IP)
  - **Save** 클릭

##### ❌ 주의사항
- ❌ **0.0.0.0 - 255.255.255.255 (모든 IP 허용)** 설정 금지
  - 보안 위험 + Azure 정책 위반 가능성

### 2.2 SSL/TLS 설정

1. 좌측 메뉴 **"TLS/SSL settings"** 클릭
2. **"Enforce SSL connection"** 설정:
   - ✅ **"Enforce SSL connection"** → **Enabled**
     - Azure App Service와 통신 시 SSL 필수
     - 보안 강화 (추가 비용 없음)

### 2.3 연결 정보 확인

1. 좌측 메뉴 **"Overview"** 클릭
2. 다음 정보 확인 및 기록:
   - **Server name**: `gateway-postgres-dev-xxxx.postgres.database.azure.com`
   - **Admin username**: `gatewayadmin` (위에서 설정한 값)
   - **Password**: (설정 시 저장한 값)

---

## 3. .NET 애플리케이션 설정 변경

### 3.1 연결 문자열 형식

Azure PostgreSQL Flexible Server 연결 문자열:

```
Host={server-name}.postgres.database.azure.com;Port=5432;Database={db-name};Username={admin-username}@{server-name};Password={password};SSL Mode=Require;Trust Server Certificate=true
```

#### 📝 예시
```
Host=gateway-postgres-dev-1234.postgres.database.azure.com;Port=5432;Database=gateway;Username=gatewayadmin@gateway-postgres-dev-1234;Password=Gateway@2024!Dev;SSL Mode=Require;Trust Server Certificate=true
```

#### 🔑 중요 포인트
- **Username**: `{admin-username}@{server-name}` 형식 (서버명 포함 필수!)
- **SSL Mode**: `Require` (Azure에서 강제)
- **Trust Server Certificate**: `true` (Azure 자체 서명 인증서 사용)

### 3.2 appsettings.Production.json 생성

`Multi-Protocol-Gateway/src/Gateway.Api/appsettings.Production.json` 파일 생성:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Host=YOUR-SERVER-NAME.postgres.database.azure.com;Port=5432;Database=gateway;Username=YOUR-ADMIN-USERNAME@YOUR-SERVER-NAME;Password=YOUR-PASSWORD;SSL Mode=Require;Trust Server Certificate=true"
  },
  "Sinks": {
    "PostgreSql": {
      "BatchSize": 100,
      "FlushIntervalMs": 1000
    },
    "JsonlFile": {
      "BasePath": "logs",
      "BatchSize": 50,
      "FlushIntervalMs": 2000
    }
  },
  "Adapters": {
    "EnableFakeAdapter": false,
    "Mqtt": {
      "Enabled": true,
      "Server": "YOUR-MQTT-BROKER-URL",
      "Port": 1883,
      "Topic": "factory/+/+/telemetry"
    }
  },
  "Serilog": {
    "Using": [ "Serilog.Sinks.Console" ],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}"
        }
      }
    ]
  },
  "AllowedHosts": "*"
}
```

⚠️ **중요**: 실제 연결 문자열로 대체 필요!

### 3.3 Azure App Service 환경 변수 설정

#### 방법 1: Azure Portal에서 설정

1. Azure Portal에서 **App Service** 리소스 선택
2. 좌측 메뉴 **"Configuration"** 클릭
3. **"Application settings"** 탭에서 **"+ New application setting"** 클릭

**추가할 설정:**
- **Name**: `ConnectionStrings__DefaultConnection`
- **Value**: 위의 연결 문자열 전체 붙여넣기
- **Deployment slot setting**: 체크 해제 (또는 필요시 체크)

4. **"Save"** 클릭 → **"Continue"** 확인
5. **⚠️ App Service 재시작 필요** (자동으로 안내됨)

#### 방법 2: Azure CLI로 설정

```bash
az webapp config connection-string set \
  --resource-group rg-gateway-dev \
  --name gateway-api-app \
  --connection-string-type PostgreSQL \
  --settings DefaultConnection="Host=YOUR-SERVER.postgres.database.azure.com;Port=5432;Database=gateway;Username=YOUR-USER@YOUR-SERVER;Password=YOUR-PASSWORD;SSL Mode=Require;Trust Server Certificate=true"
```

### 3.4 DbContext 설정 확인

현재 `Program.cs`의 DbContext 설정은 이미 올바르게 구성되어 있습니다:

```csharp
// Database
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection")
    ?? throw new InvalidOperationException("Connection string 'DefaultConnection' not found.");

var dataSourceBuilder = new NpgsqlDataSourceBuilder(connectionString);
dataSourceBuilder.EnableDynamicJson();
var dataSource = dataSourceBuilder.Build();

builder.Services.AddDbContext<GatewayDbContext>(options =>
    options.UseNpgsql(dataSource));
```

✅ **추가 변경 불필요** - 환경 변수로 연결 문자열이 자동으로 로드됨

### 3.5 Migration 적용 (로컬 → Azure DB)

#### Step 1: 로컬에서 Migration 확인

```bash
cd Multi-Protocol-Gateway/src/Gateway.Infrastructure

# Migration 목록 확인
dotnet ef migrations list

# 출력 예시:
# 20260102103731_InitialCreate
# 20260105081905_UpdateTelemetryEventSchema
```

#### Step 2: Azure DB에 Migration 적용

**방법 1: EF Core CLI 사용 (로컬에서)**

```bash
# 로컬에서 Azure DB로 직접 연결하여 Migration 적용
dotnet ef database update \
  --connection "Host=YOUR-SERVER.postgres.database.azure.com;Port=5432;Database=gateway;Username=YOUR-USER@YOUR-SERVER;Password=YOUR-PASSWORD;SSL Mode=Require;Trust Server Certificate=true"
```

⚠️ **주의**: 방화벽 규칙에 현재 PC IP가 추가되어 있어야 함

**방법 2: Azure App Service에서 자동 적용 (권장)**

`Program.cs`에 자동 Migration 코드 추가 (Production 환경에서도):

```csharp
// Ensure database is created and migrations applied
using var scope = app.Services.CreateScope();
var dbContext = scope.ServiceProvider.GetRequiredService<GatewayDbContext>();
var logger = scope.ServiceProvider.GetRequiredService<ILogger<Program>>();

try
{
    logger.LogInformation("Applying database migrations...");
    await dbContext.Database.MigrateAsync();
    logger.LogInformation("Database migrations applied successfully.");
}
catch (Exception ex)
{
    logger.LogError(ex, "An error occurred while applying database migrations.");
    throw;
}
```

⚠️ **주의**: 
- `app.Environment.IsDevelopment()` 조건 제거 (Production에서도 실행)
- 또는 `appsettings.Production.json`에 별도 플래그 추가하여 제어

**방법 3: Azure Cloud Shell에서 직접 SQL 실행**

```bash
# Azure Cloud Shell 접속 (https://shell.azure.com)
# PostgreSQL 클라이언트 설치 (없는 경우)
sudo apt-get update
sudo apt-get install postgresql-client

# 연결
psql "host=YOUR-SERVER.postgres.database.azure.com port=5432 dbname=gateway user=YOUR-USER@YOUR-SERVER password=YOUR-PASSWORD sslmode=require"

# Migration SQL 파일 직접 실행 (생성된 Migration 파일의 Up() 메서드 내용)
```

---

## 4. Azure App Service 연동 체크리스트

### ❌ DB 연결 실패 시 확인 항목 TOP 5

#### 1️⃣ 방화벽 규칙

**증상**: 
- `Npgsql.NpgsqlException: No connection could be made because the target machine actively refused it`
- `Timeout expired`

**확인 방법**:
1. Azure Portal → PostgreSQL 서버 → **Networking**
2. **Firewall rules** 섹션 확인:
   - ✅ `AllowAzureServices` 규칙이 있는지
   - ✅ App Service의 Outbound IP 주소가 추가되어 있는지

**App Service Outbound IP 확인**:
1. App Service → **Properties** 메뉴
2. **Outbound IP Addresses** 복사
3. PostgreSQL → Networking → Firewall rules에 추가

**해결책**:
```
Rule name: app-service-outbound
Start IP: {App Service의 Outbound IP 1}
End IP: {App Service의 Outbound IP 1}
```
(모든 Outbound IP를 각각 추가하거나, IP 범위로 추가)

---

#### 2️⃣ 연결 문자열 형식

**증상**:
- `Npgsql.NpgsqlException: 28P01: password authentication failed`
- `Npgsql.NpgsqlException: 3D000: database "gateway" does not exist`

**확인 방법**:
1. App Service → **Configuration** → **Application settings**
2. `ConnectionStrings__DefaultConnection` 값 확인:
   - ❌ Username에 `@server-name` 누락 여부
   - ❌ Database 이름 오타
   - ❌ Password 특수문자 이스케이프 문제

**올바른 형식**:
```
Host={server}.postgres.database.azure.com;Port=5432;Database=gateway;Username={admin}@{server};Password={password};SSL Mode=Require;Trust Server Certificate=true
```

**해결책**:
- Password에 특수문자(`@`, `;`, `=`, `%` 등) 포함 시 URL 인코딩:
  - `@` → `%40`
  - `;` → `%3B`
  - `=` → `%3D`
  - `%` → `%25`

또는 App Service Configuration에서 Value를 따옴표 없이 입력

---

#### 3️⃣ SSL 설정

**증상**:
- `Npgsql.NpgsqlException: The server does not support SSL`
- `Npgsql.NpgsqlException: Certificate validation failed`

**확인 방법**:
1. PostgreSQL → **TLS/SSL settings** 확인:
   - ✅ `Enforce SSL connection` = Enabled 인지
2. 연결 문자열 확인:
   - ✅ `SSL Mode=Require` 포함 여부
   - ✅ `Trust Server Certificate=true` 포함 여부

**해결책**:
연결 문자열에 반드시 포함:
```
SSL Mode=Require;Trust Server Certificate=true
```

---

#### 4️⃣ 포트 5432

**증상**:
- `Connection timeout`
- `No connection could be made`

**확인 방법**:
1. 연결 문자열에 `Port=5432` 명시
2. App Service에서 네트워크 아웃바운드 제한 없는지 확인

**해결책**:
- Azure PostgreSQL Flexible Server는 기본 포트 **5432** 사용
- VNet/Private Endpoint 미사용 시 포트 변경 불가

---

#### 5️⃣ App Service 로그 확인

**확인 방법**:
1. App Service → **Log stream** 메뉴
2. 실시간 로그에서 에러 메시지 확인

또는

1. App Service → **App Service logs** 메뉴
2. **Application Logging (Filesystem)** → **On** 설정
3. **Log Level** → **Verbose** 또는 **Information**
4. **Save** → 몇 분 후 **Log stream**에서 확인

**주요 에러 메시지**:
```
Npgsql.NpgsqlException: ...
Microsoft.EntityFrameworkCore.DbUpdateException: ...
System.InvalidOperationException: Connection string 'DefaultConnection' not found.
```

**해결책**:
- 에러 메시지에 따라 위 항목 1-4 재확인

---

### 📊 성공 확인 방법

1. **Health Check 엔드포인트 호출**:
   ```bash
   curl https://your-app-service.azurewebsites.net/health
   ```

2. **PostgreSQL Health Check 확인**:
   - App Service → **Health check** 메뉴
   - 또는 `/health` 엔드포인트에서 `postgresql` 상태 확인

3. **애플리케이션 로그 확인**:
   - 정상 시: `Database migrations applied successfully.`
   - 정상 시: MQTT 메시지 수신 시 `TelemetryEvents` 테이블에 데이터 저장

---

## 5. 비용 최소화 운영 팁

### 5.1 B1ms 성능 한계 및 증상

**B1ms (1 vCore / 2GB RAM) 성능 특성**:
- ✅ **CPU**: Burstable (최대 20% 평균, 피크 시 100% 사용 가능)
- ✅ **RAM**: 2GB (제한적)
- ⚠️ **동시 연결**: 약 50-100개 권장

**성능 저하 증상**:
- ❌ 쿼리 응답 시간 5초 이상
- ❌ 애플리케이션 타임아웃 발생
- ❌ `Npgsql.NpgsqlException: 53300: remaining connection slots are reserved`
- ❌ App Service 로그에 "Database connection timeout" 빈번

**권장 작업**:
- 배치 크기 줄이기 (`BatchSize: 50` → `BatchSize: 20`)
- Connection Pool 크기 조정:
  ```csharp
  builder.Services.AddDbContext<GatewayDbContext>(options =>
      options.UseNpgsql(connectionString, npgsqlOptions =>
          npgsqlOptions.MaxBatchSize(20))); // 기본값은 42
  ```

---

### 5.2 Scale-up 고려 시점

**B1ms → Standard_B2s (2 vCore / 4GB) 업그레이드 기준**:
- ✅ 동시 사용자 10명 이상
- ✅ 초당 100개 이상 Telemetry 이벤트 처리
- ✅ 쿼리 응답 시간이 계속 3초 이상
- ✅ Connection pool 고갈 빈번 발생

**비용 비교** (한국 중부 기준, 대략):
- B1ms: 약 **$12-15/월**
- B2s: 약 **$24-30/월**

**⚠️ 주의**:
- ❌ 무료 계정 기간 중에는 B1ms 유지 권장
- ❌ 필요 시에만 임시 Scale-up 후 다시 Scale-down

---

### 5.3 Storage / Backup 비용 절감

**현재 설정 (최소)**:
- Storage: 32 GB
- Backup: LRS, 7일 retention

**비용 절감 팁**:

1. **Storage 자동 증가 비활성화** (이미 설정함)
   - ❌ 무제한 자동 증가 설정 금지

2. **불필요한 데이터 정리**:
   ```sql
   -- 30일 이상 된 데이터 삭제 (예시)
   DELETE FROM telemetry_events 
   WHERE timestamp < NOW() - INTERVAL '30 days';
   ```

3. **인덱스 최적화**:
   - 불필요한 인덱스 제거 (Storage 사용량 감소)
   - 현재 설정은 적절함 (timestamp, source_id, tag 인덱스)

4. **백업 보관 기간 유지**:
   - ✅ 7일 유지 (최소값)
   - ❌ 35일로 증가 금지

---

### 5.4 개발/운영 DB 분리 전략 (비용 절감)

**현재 구성**: 단일 B1ms 인스턴스 사용

**장점**:
- ✅ 비용 1/2 (인스턴스 1개만 운영)
- ✅ Migration/스키마 관리 단순

**단점**:
- ❌ 개발 중 Production 데이터 손상 위험
- ❌ 테스트 데이터와 Production 데이터 혼재

**권장 전략**:
1. **로컬 개발**: 로컬 PostgreSQL 사용
2. **Azure 배포**: Production DB만 사용
3. **데이터 분리**: `source_id`나 `route_key`로 논리적 분리
4. **데이터 정리 스크립트**: 개발용 데이터는 주기적 삭제

---

### 5.5 ❌ 절대 하면 안 되는 비용 폭탄 설정

#### ❌ 1. Geo-redundant Backup (GRS) 선택
- **비용**: 2-3배 증가
- **대안**: LRS 사용

#### ❌ 2. Storage 무제한 자동 증가
- **비용**: 데이터 누적 시 급격히 증가
- **대안**: 최대 32-64GB로 제한

#### ❌ 3. High Availability (HA) 활성화
- **비용**: 2배 (Standby 인스턴스 추가)
- **대안**: 개발/PoC에서는 불필요

#### ❌ 4. Read Replica 생성
- **비용**: 추가 인스턴스 비용
- **대안**: 단일 인스턴스로 충분

#### ❌ 5. Private Endpoint / VNet 통합
- **비용**: 추가 네트워킹 비용 (VNet Gateway 등)
- **대안**: Public Access + 방화벽 규칙으로 충분

#### ❌ 6. Compute 자동 Scale-up 설정
- **비용**: 트래픽 증가 시 자동으로 비용 증가
- **대안**: 수동으로 필요 시에만 Scale-up

---

## 6. 최종 요약

### 6.1 예상 월 비용 (한국 중부 기준)

**PostgreSQL Flexible Server B1ms**:
- Compute (1 vCore, 2GB): 약 **$12-15/월**
- Storage (32 GB): 약 **$3-4/월** (GB당 $0.10-0.12)
- Backup (LRS, 7일): 약 **$1-2/월**
- **총 예상**: **$16-21/월** (약 20,000-25,000원)

**Azure App Service** (별도 비용):
- Basic B1 (1 vCore, 1.75GB): 약 **$13-15/월**
- 또는 Free Tier (F1): **무료** (제한적)

**총 예상 (PostgreSQL + App Service Basic B1)**:
- **$29-36/월** (약 35,000-45,000원)

⚠️ **참고**: 
- 무료 계정에는 **$200 크레딧** 포함 (12개월)
- 개발/PoC 목적이라면 첫 몇 달은 무료 또는 매우 저렴

---

### 6.2 현재 구성의 한계

#### 성능 한계
- ✅ **동시 사용자**: 약 5-10명 권장
- ✅ **초당 이벤트 처리**: 약 50-100개 권장
- ✅ **쿼리 복잡도**: 간단한 SELECT/INSERT 중심

#### 기능 한계
- ❌ **고가용성 (HA)**: 없음 (단일 인스턴스)
- ❌ **읽기 전용 복제본**: 없음
- ❌ **자동 백업 복원**: 수동 작업 필요
- ❌ **프라이빗 네트워크**: Public Access만 지원

#### 운영 한계
- ⚠️ **다운타임**: 유지보수 시 단축된 시간 (약 1-2분)
- ⚠️ **백업 보관**: 7일만 보관
- ⚠️ **데이터 손실 위험**: 단일 인스턴스이므로 높은 가용성 없음

---

### 6.3 다음 단계 (무료 기간 끝나기 전) 추천 업그레이드 경로

#### Phase 1: 무료 계정 기간 (0-12개월)
- ✅ **현재 구성 유지**: B1ms + App Service Basic B1 (또는 Free)
- ✅ 목적: 개발, PoC, 프로토타입
- ✅ 비용: 무료 또는 $20-30/월

#### Phase 2: 프로덕션 전환 시 (무료 기간 종료 전)
**옵션 A: 최소 비용 유지**
- PostgreSQL: B1ms → **B2s** (2 vCore, 4GB)
  - 비용: 약 **$24-30/월**
  - 성능: 2배 향상
- App Service: Basic B1 유지
- **총 비용**: 약 **$37-45/월**

**옵션 B: 성능 우선 (소규모 프로덕션)**
- PostgreSQL: B1ms → **General Purpose D2s_v3** (2 vCore, 8GB)
  - 비용: 약 **$80-100/월**
  - 성능: 4-5배 향상, Burstable 제한 없음
- App Service: Basic B1 → **Standard S1**
  - 비용: 약 **$55-70/월**
- **총 비용**: 약 **$135-170/월**

**옵션 C: 고가용성 추가 (중규모 프로덕션)**
- PostgreSQL: General Purpose + **HA 활성화**
  - 비용: 약 **$160-200/월** (인스턴스 2개)
- Storage 백업: **Geo-redundant (GRS)**
  - 비용: 추가 **$5-10/월**
- **총 비용**: 약 **$165-210/월**

#### Phase 3: 마이그레이션 고려사항
1. **다른 클라우드로 이동** (AWS RDS, GCP Cloud SQL)
   - 비용 비교 필요
2. **Self-hosted PostgreSQL** (VM 또는 컨테이너)
   - 관리 부담 증가, 비용 절감 가능
3. **Managed PostgreSQL 다른 서비스** (Azure Database for PostgreSQL Single Server)
   - ⚠️ Single Server는 2024년부터 신규 생성 중단

---

### ✅ 체크리스트 (최종 확인)

배포 전 확인:
- [ ] PostgreSQL Flexible Server 생성 완료 (B1ms, 32GB, LRS, 7일)
- [ ] 네트워크 설정: Public Access, Azure 서비스 허용, SSL Enforced
- [ ] 연결 문자열 생성 및 확인 (Username에 @server-name 포함)
- [ ] App Service 환경 변수 설정 완료 (`ConnectionStrings__DefaultConnection`)
- [ ] 방화벽 규칙에 App Service Outbound IP 추가
- [ ] Migration 적용 완료 (로컬 또는 App Service 자동 적용)
- [ ] Health Check 엔드포인트 정상 응답 확인
- [ ] 실제 데이터 저장 테스트 완료

---

## 📚 참고 자료

- [Azure PostgreSQL Flexible Server 가격](https://azure.microsoft.com/pricing/details/postgresql/flexible-server/)
- [Azure App Service 가격](https://azure.microsoft.com/pricing/details/app-service/windows/)
- [Npgsql 연결 문자열 가이드](https://www.npgsql.org/doc/connection-string-parameters.html)
- [EF Core Migration 가이드](https://learn.microsoft.com/ef/core/managing-schemas/migrations/)

---

**작성일**: 2026-01-06  
**작성자**: Azure Cloud Engineer  
**대상 버전**: .NET 8, PostgreSQL 15, Azure Flexible Server

