# weather-lab — LSTM 태양광 발전량 예측기

기상 데이터를 활용해 태양광 발전량을 예측하는 LSTM 시계열 모델 프로젝트입니다.  
기상청 공공 API + IoT 센서(RP2040) 데이터를 수집하고, LSTM 신경망으로 미래 발전량을 예측합니다.

---

## 프로젝트 구조

```
weather-lab/
│
├── lstm_block_sim.html      ← LSTM 블록도 신호전송 시뮬레이션 (브라우저에서 실행)
│
├── lstm_train.py            ← LSTM 모델 학습 · 평가
├── train_baseline.py        ← 비교용 베이스라인 모델 (GBM)
├── ml_shared.py             ← 데이터 전처리 · 시퀀스 생성 · 스케일러 공유 로직
│
├── collect_weather.py       ← 기상청 공공 API 데이터 수집
├── collect_weather_backfill.py  ← 과거 기상 데이터 일괄 수집
├── collect_rp2040_modbus.py ← RP2040 IoT 센서 발전량 수집 (Modbus RTU)
├── weather_public.py        ← 기상 API 호출 유틸리티
├── db.py                    ← MySQL 데이터베이스 연결
├── main.py                  ← 진입점
│
└── pyproject.toml           ← 의존성 (uv 패키지 매니저)
```

---

## 데이터 흐름

```
기상청 API / RP2040 센서
        ↓
   MySQL DB 저장
        ↓
  MinMaxScaler 정규화
        ↓
  35 타임스텝 시퀀스 생성
        ↓
    LSTM (64 units)
        ↓
    Dense (64 → 1)
        ↓
   역정규화 → 예측 발전량 (kW)
```

---

## 입력 특성 (8가지)

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

---

## 모델 구성

| 레이어 | 파라미터 |
|--------|---------|
| LSTM | 64 units, input (35, 8) |
| Dense | 1 unit (발전량 예측) |
| 옵티마이저 | Adam |
| 손실 함수 | MSE |
| 시퀀스 길이 | 35 타임스텝 (35시간 이력) |

---

## 실행 방법

```bash
# 의존성 설치 (uv)
uv sync

# 1. 베이스라인 학습
uv run python train_baseline.py

# 2. LSTM 학습
uv run python lstm_train.py
```

`.env` 파일에 DB 접속 정보를 설정해야 합니다:

```
MYSQL_HOST=127.0.0.1
MYSQL_PORT=3306
MYSQL_USER=weather
MYSQL_PASSWORD=weatherpass
MYSQL_DATABASE=weather
DEVICE_ID=RP2040-EMU-01
WEATHER_SOURCE=public
```

---

## 시뮬레이션

`lstm_block_sim.html` 을 브라우저에서 열면 LSTM 예측기의 블록도와 신호전송 과정을 시각적으로 확인할 수 있습니다.
