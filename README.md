# weather-lab — LSTM 태양광 발전량 예측기

기상청 공공 API와 RP2040 IoT 센서 데이터를 수집해 MySQL에 저장하고,  
LSTM 시계열 모델로 태양광 발전량을 예측하는 프로젝트입니다.

---

## 전체 데이터 흐름

```
기상청 공공 API (data.go.kr)
      │
      │  weather_public.py             API 호출 · 파싱
      │  collect_weather_backfill.py   DB 저장  (db.py 사용)
      ▼
  MySQL DB ◄── collect_rp2040_modbus.py ◄── RP2040 센서 (Modbus RTU/TCP)
  ├─ weather_hourly   (시간별 기상 데이터)       ※ pymysql 직접 사용
  └─ power_realtime   (시간별 발전량)
      │
      │  ml_shared.py   두 테이블 JOIN · MinMaxScaler · 시퀀스 생성
      ▼                 ※ pymysql 직접 사용
  train_baseline.py   GBM 베이스라인 학습 → metrics_baseline.json
  lstm_train.py       LSTM 학습 → 베이스라인과 MAE/RMSE 비교
```

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `weather_public.py` | 기상청 ASOS API 호출·파싱 라이브러리 |
| `collect_weather_backfill.py` | 과거 N일 기상 데이터를 DB에 일괄 저장 |
| `collect_rp2040_modbus.py` | RP2040 센서에서 Modbus RTU/TCP로 발전량 1초 주기 수집 |
| `db.py` | MySQL 연결 · 저장 함수 (`collect_weather_backfill.py`에서 사용) |
| `ml_shared.py` | DB 조회, 정규화, 시퀀스 생성, 역정규화 공유 로직 |
| `train_baseline.py` | HistGradientBoosting 베이스라인 학습 · 평가 |
| `lstm_train.py` | LSTM(64) 모델 학습 · 평가 · 베이스라인 비교 |

---

## LSTM 모델 구성

입력: 과거 **35시간** × **8가지 특성** → 출력: 다음 시각 발전량 (kW)

| 특성 | 설명 |
|------|------|
| temperature | 기온 (°C) |
| humidity | 습도 (%) |
| wind_speed | 풍속 (m/s) |
| solar_radiation | 일사량 (W/m²) |
| precipitation | 강수량 (mm) |
| power_kw | 발전량 (kW) |
| panel_temp | 패널 온도 (°C) |
| panel_humidity | 패널 습도 (%) |

```
입력 (35, 8)
    ↓  MinMaxScaler  — 각 특성을 [0, 1]로 정규화
    ↓  LSTM 64 units — 시계열 패턴 학습 → 64차원 벡터
    ↓  Dense 1       — 64 → 1 (스케일된 발전량)
    ↓  역정규화      — 실제 kW 단위로 환산
출력: 예측 발전량 (kW)
```

---

## 실행 순서

### 1. 환경 설정

`.env` 파일을 프로젝트 루트에 생성합니다.

```
MYSQL_HOST=127.0.0.1
MYSQL_PORT=3306
MYSQL_USER=weather
MYSQL_PASSWORD=weatherpass
MYSQL_DATABASE=weather

DATA_GO_KR_SERVICE_KEY=발급받은_키
ASOS_STN_ID=108

MODBUS_MODE=tcp
MODBUS_HOST=127.0.0.1
MODBUS_TCP_PORT=5020
MODBUS_SLAVE_ID=1
DEVICE_ID=RP2040-EMU-01
WEATHER_SOURCE=public
```

### 2. 의존성 설치

```bash
uv sync
```

### 3. 기상 데이터 수집 (과거 30일)

```bash
uv run python collect_weather_backfill.py 30
```

### 4. 발전량 수집 (Modbus 서버가 실행 중인 상태에서)

```bash
uv run python collect_rp2040_modbus.py
```

### 5. 베이스라인 학습

```bash
uv run python train_baseline.py
```

### 6. LSTM 학습 및 비교

```bash
uv run python lstm_train.py
```

