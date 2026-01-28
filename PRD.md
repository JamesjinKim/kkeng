# PRD: AI 기반 하수처리 무인 자동화 시스템 소프트웨어 개발

**담당 기업:** 신호테크놀로지
**문서 버전:** 1.0
**작성일:** 2025-12-01

---

## 1. 개요

### 1.1. 제품 비전

하수처리 시설의 유입수 성상을 AI로 예측하고, PSO 최적화 알고리즘을 통해 에너지 비용과 약품 비용을 최소화하면서 수질 기준을 100% 준수하는 **무인 자동운전 시스템**을 개발한다.

### 1.2. 개발 범위

신호테크놀로지는 다음 4개 레이어의 소프트웨어 개발을 담당한다:

| 레이어 | 주요 개발 내용 |
|--------|----------------|
| **Sensing Layer** | 센서 데이터 수집, 전처리, QA/QC 시스템 |
| **Analytics Layer** | Bi-LSTM 예측 모델, PSO 최적화 엔진, 검증 체계 |
| **Control Layer** | Feed-forward 제어 로직, Safety Layer, PLC 연동 |
| **Edge Infrastructure** | 엣지 AI 시스템, 데이터 압축/전송, 모니터링 |

### 1.3. 목표 처리장

- **금강엔지니어링 MBR 폐수처리시설**
- 처리 용량: 200~500 m³/일
- 기존 PLC: LS 또는 Mitsubishi
- 통신: Modbus TCP

---

## 2. 기능 요구사항

### 2.1. 센싱 레이어 (Sensing Layer)

#### 2.1.1. 데이터 수집 시스템

**FR-SEN-001: 센서 데이터 실시간 수집**
- PLC/SCADA로부터 Modbus TCP를 통해 센서 데이터 수집
- 수집 주기: 1분
- 수집 항목 (10종):

| 센서 | 측정 범위 | 정확도 | 샘플링 주기 |
|------|-----------|--------|-------------|
| pH | 0~14 | ±0.1 | 1분 |
| 수온 | -10~50°C | ±0.5°C | 1분 |
| SS (탁도계) | 0~500 mg/L | ±5% | 5분 |
| COD (UV 흡광도) | 0~200 mg/L | ±10% | 15분 |
| BOD (온라인 분석기) | 0~300 mg/L | ±15% | 1시간 |
| NH₄-N | 0~50 mg/L | ±5% | 15분 |
| DO (용존산소) | 0~20 ppm | ±0.2 mg/L | 1분 |
| 유량 | 0~500 m³/h | ±2% | 1분 |
| ORP | -500~500 mV | ±10 mV | 1분 |
| 차압계 (TMP) | 0~100 kPa | ±1% | 실시간 |

**FR-SEN-002: 데이터 전처리 파이프라인**
- 이상치 탐지: 3σ 규칙 + IQR 방법 + 물리적 범위 검증
- 결측치 처리:
  - 5분 이하: 선형 보간
  - 5~30분: Cubic Spline 보간
  - 30분 이상: 과거 동일 요일/시간대 패턴 참조
- 센서 드리프트 보정: Kalman Filter 적용
- 데이터 동기화: 1분 단위 정규화, UTC 타임스탬프

**FR-SEN-003: 데이터 저장 및 관리**
- 시계열 DB: InfluxDB 또는 TimescaleDB
- 보관 기간: 최소 3년
- 백업: 일일 자동 백업 + 월간 외부 저장
- 최소 데이터량: 500만 개 이상 (센서 10종 × 1분 주기 × 1.5년)

---

### 2.2. AI 알고리즘 레이어 (Analytics Layer)

#### 2.2.1. Bi-LSTM 예측 모델

**FR-AI-001: Control-Aware LSTM 아키텍처**

```
입력:
- 그룹 A (시계열): 72시간 × 25개 features
  - 센서 직접 측정값: 15개
  - 파생 변수: 7개 (F/M비, 유기물 부하율, HRT, SRT, 송풍기 가동률, 반송슬러지 비율, 누적 강우량)
  - 시간 변수: 3개 (시간대, 요일, 계절)
- 그룹 B (제어 변수): 4개
  - 송풍량 설정값 (0~50 m³/min)
  - PAC 투입량 (0~200 mg/L)
  - Polymer 투입량 (0~10 mg/L)
  - NaOH 투입량 (0~50 L/hr)

모델 구조:
- Bi-LSTM Layer 1: 128 units, Dropout 0.3
- Bi-LSTM Layer 2: 64 units, Dropout 0.3
- Self-Attention Mechanism
- Dense Layer: 64 units (Sensor + Control 융합)
- Output: 5개 (COD, BOD, SS, T-N, T-P)

출력:
- 6시간 후 방류수 수질 예측
```

**FR-AI-002: 학습 전략**
- 데이터 분할: Train 70% / Validation 15% / Test 15%
- Optimizer: Adam (lr=0.001)
- Loss: Huber Loss
- Batch Size: 128
- Epochs: 200 (Early Stopping patience=20)
- Regularization: L2 (λ=0.001) + Dropout 0.3

**FR-AI-003: 성능 목표**

| 지표 | 목표치 |
|------|--------|
| MAPE | < 10% |
| R² Score | > 0.85 |
| 방향 정확도 | > 90% |
| 추론 시간 | < 2초 (엣지) |

#### 2.2.2. PSO 최적화 엔진

**FR-PSO-001: PSO 최적화 알고리즘**
- 최적화 대상:
  - 약품 투입량 (PAC, Polymer, NaOH)
  - 펌프 운전 스케줄 (24시간)
  - 터보 브로워 속도 (0~100%)
  - 분리막 운전 사이클 (흡입/휴지 시간)

- 목적함수:
```
minimize: α₁ × 전력비용 + α₂ × 약품비용 + α₃ × 수질위반_페널티
가중치: α₁=0.7, α₂=0.2, α₃=0.1
```

- 제약조건:
  - COD ≤ 40 mg/L (법적 기준)
  - BOD ≤ 10 mg/L
  - SS ≤ 10 mg/L
  - T-N ≤ 20 mg/L
  - T-P ≤ 2 mg/L

**FR-PSO-002: 경량화 PSO (엣지 최적화)**
- 입자 수: 20개 (기존 100개에서 축소)
- 세대 수: 15세대 (기존 50세대에서 축소)
- Early Stopping: 3세대 동안 개선 없으면 중단
- 배치 추론: 20개 입자 동시 평가
- 목표 실행 시간: 45초 이내 (30분 주기)

**FR-PSO-003: 실행 주기**
- 정상 운전: 30분 주기
- 부하 급변 감지: 즉시 실행 (5분 내)
- 긴급 상황: Rule-based 제어로 전환 (PSO 우회)

#### 2.2.3. 검증 체계 (3-Tier)

**FR-VAL-001: Tier 1 실시간 센서 검증 (1분 주기)**
- COD/BOD 비율 검증 (정상: 1.2~2.5)
- DO-송풍량 상관관계 검증
- pH 급격 변화 감지 (1분간 0.5 이상 변화 시 경보)
- 질량 보존 검증 (유입 유량 ≈ 유출 유량)

**FR-VAL-002: Tier 2 온라인 분석기 검증 (일 1회)**
- 온라인 COD/BOD 분석기 vs AI 예측값 비교
- MAPE > 15% 시 재보정 필요 플래그
- 일일 리포트 자동 생성

**FR-VAL-003: Tier 3 실험실 정밀 분석 (주 2회)**
- 실험실 분석 결과와 AI 예측 비교
- 월간 MAPE > 12% 시 모델 재학습 트리거
- 자동 재학습 파이프라인

---

### 2.3. 제어 레이어 (Control Layer)

#### 2.3.1. Feed-forward 제어 로직

**FR-CTL-001: 부하 기반 제어 알고리즘**

```
IF 유입 BOD 예측값 > 200 mg/L (고부하):
  - 송풍기 RPM: +25%
  - 반송슬러지 비율: 50% → 75%
  - 체류시간: 8h → 10h

ELSE IF 유입 BOD 예측값 < 80 mg/L (저부하):
  - Eco Mode 전환
  - 송풍기 RPM: -20%
  - 간헐 포기 운전

ELSE (정상):
  - DO 목표값: 2.0~2.5 mg/L 유지

추가 규칙:
- 수온 < 10°C: 송풍량 +15% 보정
- 강우 예측 (6h 내 20mm 이상): 유입조 수위 사전 확보
- NH₄-N > 30 mg/L: 질산화 강화 모드 (DO 3.5 mg/L)
```

**FR-CTL-002: 하이브리드 제어 (Feed-forward + Feedback)**
- 주 제어: Feed-forward (6시간 선행)
- 보조 제어: Feedback (실시간 미세 조정)
- 전환 로직:
  - |방류수 실측 - 예측| > 20%: Feedback 가중치 증가
  - 방류수 COD > 법적 기준의 80%: 긴급 제어

#### 2.3.2. 인터락 제어 시스템

**FR-CTL-003: 수위 기반 인터락**

| Tank | 조건 | 동작 |
|------|------|------|
| 분리막조 | L (저수위) | 흡입펌프 정지 |
| 분리막조 | HH (고수위) | 1차처리수조 펌프 정지 |
| 분리막조 | HHAL (초고수위) | 경보 + 유량조정조 펌프 정지 |
| 1차처리수조 | HH | 유량조정조 펌프 정지 |
| 여과수조 | H | 흡입펌프 정지 |
| 방류조 | HH | 여과수조 펌프 정지 |

**FR-CTL-004: 가압부상조(DAF) 연동 시퀀스**

```
유량조정조 펌프 ON 시:
Step 1: SOL-101 밸브 열림
Step 2: 순환수 가압펌프 가동
Step 3: PAC 약품펌프 가동
Step 4: 반응조/pH조정조/응집조 교반기 가동
Step 5: 폴리머 약품펌프 가동

유량조정조 펌프 OFF 시:
역순 정지 (지연 시간 적용)
```

**FR-CTL-005: pH 제어 인터락**
- 유량조정조 펌프 ON + pH < LOW: NaOH 펌프 가동
- pH > HIGH: NaOH 펌프 정지
- pH 조정조 교반기 정지: NaOH 펌프 연동 정지

**FR-CTL-006: 과전류 보호**
- A 펌프 과전류 → 자동 정지 → B 펌프 자동 기동
- FAULT 경보 발생 + PLC 로그 기록

**FR-CTL-007: 분리막 보호**
- 흡입펌프 연속 가동 > 7분: 자동 정지 (막 폐쇄 방지)
- 차압 > 설정값: 주간세정 필요 알림
- 차압 >> 고설정값: 인라인세정 경보

#### 2.3.3. Safety Layer (6층 검증)

**FR-SAF-001: 물리적 제약 검증**
- 펌프 용량 초과 여부 (정격 대비 80% 이내)
- 약품 잔량 충분 여부 (최소 3일분)
- 설비 연속 가동 시간 한도

**FR-SAF-002: 예측 신뢰도 검증**
- Uncertainty > 0.3: PSO 결과를 보수적으로 조정 (안전 계수 ×1.2)

**FR-SAF-003: 법적 기준 사전 확인**
- 예상 방류수질 > 법적 기준의 80%: 송풍량 강제 증가 + 운영자 경고

**FR-SAF-004: 센서 이상 대응**
- 주요 센서 고장: 백업 센서 또는 추정값 사용
- 복수 센서 동시 이상: 수동 모드 전환

#### 2.3.4. 제어 모드 전환

| 모드 | 조건 | Bi-LSTM | PSO | 송풍기 |
|------|------|---------|-----|--------|
| **정상 AI** | 수질 안정, 신뢰도 높음 | 5분 주기 | 30분 주기 | PSO 최적값 |
| **안전** | 이상 감지, 센서 불확실 | 10분 주기 | 중단 | DO 2.5 mg/L 고정 |
| **비상** | 수질 기준 초과 위험 | 중단 | 중단 | 최대 가동 |
| **Eco** | 저부하, 야간 | 30분 주기 | 1시간 주기 | 간헐 포기 |

**모드 전환 조건:**

```
정상 AI → 안전:
- Anomaly Score > 0.7
- 예측 불확실성 > 0.3
- 센서 2개 이상 이상

안전 → 비상:
- 방류 COD > 36 mg/L (법적 90%)
- 터보 브로워 정지
- TMP > 50 kPa

비상 → 안전:
- 방류수질 < 법적 70% 회복
- 설비 정상 확인

안전 → 정상 AI:
- 24시간 연속 안정 운전
- MAPE < 10%
- 센서 모두 정상
```

---

### 2.4. 엣지 인프라 레이어 (Edge AI Infrastructure)

#### 2.4.1. 3-Tier 하이브리드 아키텍처

> **아키텍처 결정 근거**: AI 추론을 Tier 2 (Gateway)에서 수행
> - 데이터 흐름: Tier 1 수집 → Tier 2 InfluxDB 저장 후 AI 추론 → Tier 3 제어
> - 모델 관리/디버깅/업데이트 용이 (중앙 집중화)
> - Tier 1 비용 절감 (Hailo NPU 불필요, ~$100 절약)
> - Fallback: 네트워크 단절 시 Tier 1에서 Rule-based로 Tier 3 직접 제어

**FR-EDGE-001: Tier 1 엣지 디바이스 (현장 데이터 수집)**

하드웨어:
- Raspberry Pi 5 (4GB RAM) - 데이터 수집에 충분
- 128GB microSD
- ADC/I2C/SPI 센서 인터페이스
- 산업용 DIN 레일 케이스 (IP65)
- 전력: 3~5W
- 비용: ~$150

기능:
- 센서 데이터 수집: 1분 주기 (Modbus TCP, ADC)
- QA/QC 전처리: 이상치 탐지, 결측치 보간
- 로컬 버퍼: 24시간 데이터 캐시 (네트워크 복구 대비)
- MQTT 전송: Tier 2로 1분 주기 데이터 전송
- **Fallback 제어**: 네트워크 단절 시 Rule-based로 Tier 3 직접 제어
  - DO < 1.5 mg/L: 송풍기 RPM +20%
  - pH < 6.5: NaOH 펌프 ON
  - 수위 > HH: 유입펌프 OFF
  - 기타: 마지막 정상 제어값 유지

**FR-EDGE-002: Tier 2 게이트웨이 (AI 추론 + 데이터 관리)**

하드웨어:
- Mini PC/NUC (8GB RAM, i5급)
- Hailo AI Kit (Hailo-8L NPU, 13 TOPS)
- 256GB SSD
- 비용: ~$300 (Mini PC $200 + Hailo $70 + 기타)

기능:
- 데이터 저장: InfluxDB (장기 보관, 최소 3년)
- **Bi-LSTM 예측**: 5분 주기, 추론 시간 1~2초 (Hailo NPU 가속)
- **PSO 최적화**: 30분 주기, 45~90초
- **Safety Layer**: 6층 검증 (실시간)
- Dashboard: Grafana 기반 모니터링
- OTA 업데이트: 모델/설정 원격 배포
- 다중 현장 지원: 5~10개 Tier 1 디바이스 관리 가능

**FR-EDGE-003: Tier 3 제어 계층 (액추에이터 제어)**

하드웨어:
- ESP32/Arduino (산업용 모듈)
- 솔리드스테이트 릴레이 모듈
- Safety PLC (핵심 인터락용, 최소화)
- 비용: ~$100

기능:
- 제어 대상: 송풍기, 펌프, 약품투입기, 밸브
- 운전 모드:

| 모드 | 설명 | AI 역할 |
|------|------|---------|
| **무인자동** | AI Feed-forward 제어, PSO 최적값 자동 적용 | 전적 제어 |
| **반자동** | AI 권장 후 운영자 승인 시 실행 | 권장 제공 |
| **수동** | 운영자 직접 제어 | 모니터링만 |

- 통신: Tier 2 → Serial/GPIO (제어 명령)
- 안전장치: Safety PLC가 핵심 인터락 (수위 HH, 과전류) 독립 처리

**FR-EDGE-004: 클라우드 (장기 목표)**
- 전국 데이터 통합 분석
- 대형 모델 재학습 (주/월 단위)
- Transfer Learning (신규 시설 적용)
- 통계 분석 및 BI 리포트
- 현재 구현 범위에서 보류, 추후 확장

#### 2.4.2. AI 모델 최적화

**FR-EDGE-004: 모델 변환 파이프라인**

```
TensorFlow (h5)
    ↓ tf2onnx
ONNX
    ↓ Hailo Dataflow Compiler
Hailo HEF (INT8 양자화)

결과:
- 모델 크기: 4,500 KB → 1,200 KB (73% 감소)
- 추론 속도: 3초 → 1.2초 (2.5배 향상)
- 정확도 손실: MAPE 9.2% → 9.5% (<2%)
```

**FR-EDGE-005: 경량화 PSO 구현**
- 스마트 초기화: 현재 제어값 ±30% 범위에서 입자 생성
- 배치 추론: 20개 입자 동시 평가 (3초/배치)
- Early Stopping: 3세대 개선 없으면 중단
- 실행 시간: 45초 (30분 주기에 충분)

#### 2.4.3. 데이터 압축 및 전송

**FR-EDGE-006: 3단계 데이터 전송**

| 구간 | 주기 | 원본 | 전송 | 압축률 |
|------|------|------|------|--------|
| Edge → Gateway | 10분 | 2,000B | 800B | 60% |
| Gateway → Cloud | 1시간 | 28,800B | 1,200B | 96% |
| 일일 (1개 현장) | - | 2.7GB | 28.8MB | 99% |

전송 내용 (10분 주기):
- 센서 데이터 요약 (avg, min, max)
- AI 예측 결과 (COD, BOD, SS, confidence)
- 제어 파라미터 (송풍기, 약품, 에너지)
- KPI (에너지 비용, 약품 비용, 수질 점수)
- 이상 이벤트 (발생 시만)

#### 2.4.4. 환경 내구성 대응

**FR-EDGE-007: 산업용 인클로저**
- 등급: IP68 + NEMA 4X (방진/방수/부식방지)
- HEPA 필터 환기팬
- 열전도 패드, 방진 마운트
- 실리카겔 제습제, VCI 부식방지 패드

**FR-EDGE-008: 환경 모니터링**
- CPU 온도 > 75°C: 팬 부스트
- 습도 > 85%: 제습제 교체 알림
- 온도 범위 이탈 (0°C~50°C): 안전 모드 진입

**FR-EDGE-009: N+1 이중화**
- Active-Standby 구성
- Heartbeat 모니터링 (10초 주기)
- 장애 감지 시 Failover (< 60초)

---

## 3. 비기능 요구사항

### 3.1. 성능 요구사항

| 항목 | 목표치 |
|------|--------|
| Bi-LSTM 추론 시간 | < 2초 |
| PSO 최적화 시간 | < 120초 |
| PLC 제어 명령 지연 | < 10ms |
| 시스템 가동률 | > 99% |
| 데이터 전송 압축률 | > 95% |

### 3.2. 확장성 요구사항

- 단일 게이트웨이: 5~10개 현장 관리
- OTA 업데이트: 원격 모델 배포
- Transfer Learning: 신규 시설 적용 지원

### 3.3. 보안 요구사항

- 민감 데이터 현장 내 처리 (엣지)
- HTTPS/TLS 암호화 통신
- 인증된 사용자만 제어 명령 가능

### 3.4. 유지보수 요구사항

| 주기 | 항목 |
|------|------|
| 주 1회 | CPU 온도 로그 확인 |
| 월 1회 | 인클로저 외관 점검, 제습제 교체 |
| 분기 1회 | 커넥터 접점 확인, SD 카드 건강 상태 |
| 연 1회 | microSD 카드 교체 (산업 환경) |

연간 유지보수 비용: ~$72 (약 10만원)

---

## 4. 기술 스택

### 4.1. AI/ML

| 구분 | 기술 |
|------|------|
| 딥러닝 프레임워크 | TensorFlow 2.15 |
| 모델 변환 | ONNX, Hailo SDK |
| NPU 가속 | Hailo-8L (13 TOPS) |
| 최적화 알고리즘 | PSO (Particle Swarm Optimization) |

### 4.2. 데이터

| 구분 | 기술 |
|------|------|
| 시계열 DB | InfluxDB / TimescaleDB |
| 로컬 DB | SQLite |
| 데이터 분석 | NumPy, Pandas, SciPy |

### 4.3. 통신

| 구분 | 기술 |
|------|------|
| PLC 통신 | Modbus TCP/RTU (pymodbus) |
| 게이트웨이 통신 | MQTT (paho-mqtt) |
| 클라우드 통신 | HTTPS REST API |

### 4.4. 플랫폼

| 구분 | 기술 |
|------|------|
| 엣지 OS | Raspberry Pi OS (64-bit) |
| 컨테이너 | Docker (옵션) |
| 배포 자동화 | Ansible |

### 4.5. 시각화

| 구분 | 기술 |
|------|------|
| 로컬 대시보드 | Flask, Dash, Plotly |
| 모니터링 | Grafana |

### 4.6. 프로젝트 디렉토리 구조

```
ec-mbr-ai/
├── README.md                    # 프로젝트 개요
├── docs/                        # 문서
│   ├── PRD.md
│   ├── TechArc.md
│   └── operation_table.md
│
├── tier1-edge/                  # Tier 1: 데이터 수집 (RPi)
│   ├── src/
│   │   ├── collector/           # 센서 데이터 수집
│   │   │   ├── modbus_client.py
│   │   │   ├── adc_reader.py
│   │   │   └── sensor_config.yaml
│   │   ├── qaqc/                # 전처리 (이상치, 결측치)
│   │   │   ├── outlier_detector.py
│   │   │   └── interpolator.py
│   │   ├── buffer/              # 로컬 캐시 (24시간)
│   │   │   └── local_buffer.py
│   │   ├── mqtt/                # Tier 2 통신
│   │   │   └── publisher.py
│   │   └── fallback/            # 네트워크 단절 시 Rule-based 제어
│   │       └── rule_controller.py
│   ├── config/
│   │   └── settings.yaml
│   ├── tests/
│   └── requirements.txt
│
├── tier2-gateway/               # Tier 2: AI 추론 + 데이터 관리
│   ├── src/
│   │   ├── ai/                  # AI 모델
│   │   │   ├── bilstm/
│   │   │   │   ├── model.py
│   │   │   │   ├── predictor.py
│   │   │   │   └── attention.py
│   │   │   ├── pso/
│   │   │   │   ├── optimizer.py
│   │   │   │   └── objective.py
│   │   │   └── hailo/           # NPU 최적화
│   │   │       ├── converter.py
│   │   │       └── runtime.py
│   │   ├── safety/              # Safety Layer (6층 검증)
│   │   │   ├── physical_limits.py
│   │   │   ├── confidence_check.py
│   │   │   ├── legal_compliance.py
│   │   │   ├── sensor_anomaly.py
│   │   │   ├── electrode_protection.py
│   │   │   └── membrane_protection.py
│   │   ├── db/                  # 데이터베이스
│   │   │   ├── influxdb_client.py
│   │   │   └── schemas.py
│   │   ├── mqtt/                # Tier 1 수신
│   │   │   └── subscriber.py
│   │   ├── control/             # Tier 3 명령 전송
│   │   │   └── command_sender.py
│   │   └── dashboard/           # Grafana 연동
│   │       └── grafana_config.json
│   ├── models/                  # 학습된 모델 저장
│   │   ├── bilstm_v1.h5
│   │   └── bilstm_v1.hef        # Hailo 포맷
│   ├── config/
│   ├── tests/
│   └── requirements.txt
│
├── tier3-control/               # Tier 3: 액추에이터 제어 (ESP32)
│   ├── firmware/
│   │   ├── main/
│   │   │   ├── main.c
│   │   │   ├── actuator_control.c
│   │   │   ├── relay_driver.c
│   │   │   └── mode_manager.c   # 무인자동/반자동/수동
│   │   └── CMakeLists.txt
│   ├── include/
│   │   └── config.h
│   └── README.md
│
├── shared/                      # 공용 모듈
│   ├── protocols/               # 통신 프로토콜 정의
│   │   ├── mqtt_topics.py
│   │   └── command_schema.py
│   ├── constants/               # 상수 (법적 기준, 센서 범위 등)
│   │   ├── legal_limits.py
│   │   └── sensor_ranges.py
│   └── utils/
│       └── logger.py
│
├── scripts/                     # 유틸리티 스크립트
│   ├── deploy/                  # 배포 스크립트
│   │   ├── tier1_setup.sh
│   │   ├── tier2_setup.sh
│   │   └── ansible/
│   ├── training/                # 모델 학습
│   │   └── train_bilstm.py
│   └── simulation/              # 시뮬레이션
│       └── plant_simulator.py
│
├── docker/                      # Docker 설정
│   ├── tier2/
│   │   └── Dockerfile
│   └── docker-compose.yml
│
└── .github/                     # CI/CD
    └── workflows/
        └── test.yml
```

#### 개발 우선순위

| 순위 | 모듈 | Tier | 의존성 |
|------|------|------|--------|
| 1 | `tier1-edge/collector` | 1 | 없음 (독립 개발 가능) |
| 2 | `tier2-gateway/db` | 2 | Tier 1 데이터 필요 |
| 3 | `tier2-gateway/ai/bilstm` | 2 | DB 데이터 필요 |
| 4 | `tier2-gateway/ai/pso` | 2 | Bi-LSTM 예측 필요 |
| 5 | `tier2-gateway/safety` | 2 | PSO 출력 필요 |
| 6 | `tier3-control/firmware` | 3 | Tier 2 명령 필요 |
| 7 | `tier1-edge/fallback` | 1 | Tier 3 연동 후 개발 |

---

## 5. 인터페이스 명세

### 5.1. PLC 통신 (Modbus TCP)

```
프로토콜: Modbus TCP
포트: 502
레지스터 맵:
  - Holding Registers: 센서 데이터 읽기
  - Coils: 펌프/밸브 ON/OFF 제어
  - Input Registers: 설비 상태 읽기
```

### 5.2. 게이트웨이 통신 (MQTT)

```
브로커: Mosquitto
토픽 구조:
  - wastewater/site/{site_id}/sensors: 센서 데이터
  - wastewater/site/{site_id}/prediction: AI 예측
  - wastewater/site/{site_id}/control: 제어 명령
  - wastewater/alert: 이상 이벤트
```

### 5.3. 클라우드 API (REST)

```
Base URL: https://cloud.example.com/api/v1

엔드포인트:
  POST /hourly: 시간별 집계 데이터 전송
  POST /alert: 이상 이벤트 즉시 전송
  GET /model/{version}: 모델 업데이트 다운로드
  GET /config: 설정값 동기화
```

---

## 6. 개발 일정

### Phase 1: 엣지 프로토타입 개발 (2개월)

| 주차 | 작업 내용 |
|------|-----------|
| 1 | 하드웨어 설정 (RPi + AI Kit), PLC 시뮬레이터 연결 |
| 2-3 | AI 모델 최적화 (TensorFlow → ONNX → Hailo HEF) |
| 4 | PLC 연동 테스트 (Modbus TCP) |
| 5-6 | Safety Layer 구현, 시뮬레이션 테스트 |
| 7 | Gateway 연동, OTA 업데이트 테스트 |
| 8 | 72시간 연속 운전 시뮬레이션, 성능 검증 |

산출물: 엣지 시스템 v1.0, 벤치마크 리포트, 설치 매뉴얼

### Phase 2: 현장 파일럿 테스트 (4개월)

| 주차 | 작업 내용 |
|------|-----------|
| 9 | 현장 설치 (PLC 제어반 DIN 레일 마운트) |
| 10-11 | 모니터링 모드 (AI 예측만, 실제 제어 없음) |
| 12-13 | 반자동 모드 (AI 권장 + 운영자 승인) |
| 14-17 | 자동 모드 (주간 완전 자동, 야간 비상 대기) |

산출물: 72시간 무인 운전 성공, 에너지 절감 15% 달성, 실증 보고서

### Phase 3: 성능 최적화 및 확장 (2개월)

| 월 | 작업 내용 |
|----|-----------|
| 5 | 현장 데이터 Fine-tuning, 계절별 특성 반영 |
| 6 | Transfer Learning 매뉴얼, 10개 현장 확장 계획 |

산출물: AI 모델 v2.0, 기술 이전 가이드, 확장 계획서

---

## 7. 성공 지표 (KPI)

| 지표 | 목표치 | 측정 방법 |
|------|--------|-----------|
| 방류수질 기준 초과 | 0회/년 | TMS 모니터링 |
| 에너지 절감률 | > 15% | 전력량계 비교 |
| 예측 정확도 (MAPE) | < 10% | 실측값 vs 예측값 |
| 무인 운전 시간 | > 72시간 | 운영자 개입 기록 |
| AI 시스템 가동률 | > 99% | 시스템 로그 |
| 투자 회수 기간 | < 45일 | TCO 분석 |

---

## 8. 비용 분석

### 8.1. 하드웨어 비용 (1개 현장)

| 항목 | 비용 |
|------|------|
| Raspberry Pi 5 (8GB) | $80 |
| Hailo AI Kit | $70 |
| 128GB microSD | $20 |
| 전원 어댑터 | $12 |
| 산업용 케이스 (IP65) | $75 |
| **소계 (필수)** | **$257** |
| PoE HAT, SSD, UPS, Modbus 변환기 | $125 |
| **총계** | **$382** |

### 8.2. 3년 TCO 비교

| 항목 | 클라우드 전용 | 엣지 + 클라우드 |
|------|--------------|----------------|
| 초기 투자 | $0 | $457 |
| 월 운영비 | $338 | $23 |
| 3년 총비용 | $12,168 | $1,285 |
| **절감액** | - | **$10,883 (89%)** |

ROI: $457 / $315 = **1.45개월 (약 45일)**

---

## 9. 리스크 및 대응

| 리스크 | 영향 | 대응 방안 |
|--------|------|-----------|
| 모델 예측 오류 | 수질 기준 위반 | Safety Layer 6층 검증, Fallback 전략 |
| PSO 수렴 실패 | 불안정한 제어 | 경량화 PSO, Early Stopping, 이전 최적값 유지 |
| 센서 고장 | 잘못된 입력 | 이중화 센서, 물리적 관계 검증, 추정값 사용 |
| 엣지 장비 고장 | 제어 중단 | N+1 이중화, 자동 Failover (<60초) |
| 네트워크 장애 | 원격 모니터링 불가 | 엣지 독립 운전, 로컬 7일 데이터 보관 |
| 산업 환경 내구성 | 장비 수명 단축 | IP68 인클로저, 환경 모니터링, 예방적 유지보수 |

---

## 10. 용어 정의

| 용어 | 설명 |
|------|------|
| Bi-LSTM | Bidirectional Long Short-Term Memory |
| PSO | Particle Swarm Optimization (입자군 최적화) |
| Feed-forward | 유입수 기반 사전 제어 방식 |
| Feedback | 방류수 기반 사후 제어 방식 |
| TMP | Trans-Membrane Pressure (막간압력차) |
| DO | Dissolved Oxygen (용존산소) |
| MAPE | Mean Absolute Percentage Error |
| HEF | Hailo Executable Format |
| OTA | Over-The-Air (원격 업데이트) |
| TCO | Total Cost of Ownership |

---

## 부록 A: 장비 운전 조건 및 인터락 상세

### A.1. 펌프류 (10종)

| No. | 장비명 | 사양 | 운전 조건 | 인터락 |
|-----|--------|------|-----------|--------|
| 1 | 흡입펌프(A/B) 기존막 | 40A × 0.12㎥/min × 0.6kW | 분리막조 HI + 타이머 (7분 ON / 2분 OFF) | LOW→정지, 여과수조 H→정지, 7분 초과→자동정지 |
| 2 | 흡입펌프(C/D) 신규막 | 50A × 0.2㎥/min × 1.7kW | 동일 | 동일 |
| 3 | 유량조정조 펌프(A/B) | 80A × 0.5㎥/min × 5.5kW | 유량조정조 HI | 1차처리수조 HH→정지, DAF 자동 연동 |
| 4 | 1차처리수조 펌프(A/B) | 50A × 0.2㎥/min × 1.5kW | 1차처리수조 H | 분리막조 HH→정지 |
| 5 | 여과수조 펌프(A/B) | 80A × 0.2㎥/min × 7.5kW | 여과수조 H | 방류조 HH→정지 |
| 6 | 슬러지 이송펌프(A/B) | 50A × 0.2㎥/min × 1.5kW | 탈수기 판넬 연동 | 탈수기 정지→연동 정지 |
| 7 | 여액 이송펌프(A) | 50A × 0.1㎥/min × 1.5kW | 여액조 H | - |
| 8 | 반송펌프(A/B) | 50A × 0.2㎥/min × 1.5kW | PLC 타이머 | - |
| 9 | 방류펌프(A/B) | 80A × 0.2㎥/min × 2.2kW | 방류조 H | - |
| 10 | 순환수 가압펌프(A/B) | 65A × 0.3㎥/min × 5.09kW | 유량조정조 펌프 연동 | - |

### A.2. 약품 투입 펌프류 (5종)

| No. | 장비명 | 사양 | 운전 조건 |
|-----|--------|------|-----------|
| 11 | PAC 공급 펌프(A/B) | 1,020cc/min × 0.2kW | 유량조정조 펌프 연동 |
| 12 | 폴리머 공급 펌프(A/B) | 1,020cc/min × 0.2kW | 유량조정조 펌프 연동 |
| 13 | 가성소다 공급 펌프(A/B) | 1,020cc/min × 0.2kW | 유량조정조 펌프 ON + pH LOW |
| 14 | 세정약품 공급 펌프(A/B) | 1,020㎖/min × 0.2kW | PLC 역세 시간 |
| 15 | 역세펌프 | 25A × 0.03㎥/min × 0.25kW | 역세 시 약품 이송 |

### A.3. 송풍/교반 장비류 (8종)

| No. | 장비명 | 사양 | 비고 |
|-----|--------|------|------|
| 16 | 터보 브로워(A/B) | 22.0㎥/min × 21.7kW | 전력의 50.4% (AI 최적화 핵심 대상) |
| 17 | 반응조 교반기 | 120RPM × 3.7kW | 유량조정조 펌프 OFF 후 60초 지연 정지 |
| 18 | pH조정조 교반기 | 180RPM × 3.7kW | 가성소다 펌프 OFF 시 연동 정지 |
| 19 | 응집조 교반기 | 60RPM × 3.7kW | 유량조정조 펌프 연동 |
| 20-23 | 약품탱크 교반기 | 180RPM × 0.75~2.25kW | 간헐/역세 연동 |

---

**문서 끝**
