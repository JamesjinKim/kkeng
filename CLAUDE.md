# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a research planning repository for developing an **AI-based Autonomous Wastewater Treatment System** using Feed-forward control. The project aims to implement unmanned wastewater treatment automation by predicting influent water quality and proactively controlling treatment processes.

**Core Technologies:**
- **Bidirectional LSTM with Attention**: Time-series prediction of water quality parameters (COD, BOD, SS, DO, pH)
- **Particle Swarm Optimization (PSO)**: Optimization of control parameters (chemical dosing, pump schedules, blower speed, membrane operation)
- **Feed-forward Control**: Proactive control based on influent water quality predictions (vs. traditional reactive feedback control)

**Target Implementation:**
- Gold Gang Engineering (금강엔지니어링) MBR wastewater treatment facility
- Processing capacity: 200-500 m³/day
- Integration with existing PLC-based SCADA systems

## Repository Structure

### Core Documentation Files

- **`project.md`**: Complete R&D proposal document (Korean)
  - Research background, objectives, methodology
  - Sensor network specifications based on actual operational data
  - AI model architecture (Bi-LSTM with Attention)
  - Feed-forward control logic and interlock specifications
  - 12-month implementation roadmap
  - Budget breakdown and economic analysis

- **`operation_table.md`**: Operation sequence table
  - PLC control sequences for each system/tank
  - Interlock conditions and safety protocols
  - Level sensor triggers and automated responses
  - Based on actual operational manual from Gold Gang Engineering

- **`AI적용방법.md`**: AI implementation methodology
  - Integration architecture combining Bi-LSTM and PSO
  - Real-time operational flow (5 steps)
  - System-specific applications (DAF, Membrane Tank, etc.)
  - Phase-by-phase implementation roadmap
  - Risk management and safety protocols

- **`presentation.html`**: Research proposal presentation (20 slides)
  - HTML-based slide deck for stakeholder presentations
  - Includes economic analysis, technical specifications, and expected outcomes
  - Optimized for PDF export with print styles

- **`TechArc.md`**: EC-MBR AI 시스템 기술 아키텍처
  - 4-Layer 아키텍처 (Sensing → Analytics → Control → Edge Infrastructure)
  - EC-MBR 특화 모듈 (전극 수명 예측, 동적막 최적화, 전기응집-MBR 통합 제어)
  - Feed-forward 6시간 선행 예측 제어 상세 구현
  - Safety Layer 6층 검증 체계

- **`PRD.md`**: 소프트웨어 개발 PRD (신호테크놀로지 담당)
  - 기능 요구사항 (FR-SEN, FR-AI, FR-PSO, FR-CTL, FR-SAF, FR-EDGE)
  - 기술 스택 명세 (TensorFlow, InfluxDB, Modbus TCP, MQTT)
  - 하드웨어 사양 (Raspberry Pi 5 + Hailo-8L NPU)
  - 장비별 인터락 상세 (부록 A)

- **`각조및 장비의 기능표.md`**: 처리시설 장비 기능표
  - 10개 조(Tank) 역할 및 유지관리 포인트
  - 펌프/교반기/송풍기 사양 및 운전 조건

### Supporting Files

- **`폐수처리현장 운영 매뉴얼_금강엔지니어링(주).pdf`**: Actual operational manual from the target facility
- **`AI 기반 하수처리 무인 자동화 시스템 개발.pdf`**: Full research proposal PDF
- **`project_presentation.pdf/pptx`**: Presentation materials

## Key Technical Concepts

### 1. System Architecture

```
Sensor Data Collection (1-min cycle)
    ↓
Bi-LSTM with Attention Prediction (5-min cycle)
    ↓
Situation Assessment
    ↓
PSO Optimization (30-min cycle)
    ↓
PLC Control Execution
```

### 2. Wastewater Treatment Process Flow

```
Influent → Flow Equalization Tank → Reaction Tank (PAC) → pH Adjustment (NaOH)
→ Coagulation Tank (Polymer) → DAF System → 1st Treatment Tank
→ Aeration Tanks (1-4) → Membrane Tank (MBR) → Filtrate Tank
→ A/C Filter → Discharge Tank
```

### 3. AI Model Specifications

**Bi-LSTM Model:**
- **Input**: 25 variables (15 sensor data + 7 derived parameters + 3 temporal features)
- **Output**: 5 predictions (6-hour ahead COD, BOD, SS, T-N + optimal blower airflow)
- **Performance Targets**: MAPE < 10%, R² > 0.85, Directional accuracy > 90%

**PSO Optimization:**
- **Objective**: Minimize (chemical cost + energy cost) while maintaining water quality
- **Variables**: Chemical dosing (PAC, Polymer, NaOH), pump schedules, blower RPM, membrane operation timing
- **Constraints**: Legal discharge standards, equipment limits, predicted water quality thresholds

### 4. Critical Interlock Conditions

Based on actual PLC sequences documented in `operation_table.md`:

- **Membrane Tank HHAL**: Stop 1st Treatment Pump + Alarm
- **1st Treatment Tank HH**: Stop Flow EQ Pump
- **Flow EQ Pump ON**: Auto-start DAF System (SOL valve → Circulation pump → Chemical pumps → Agitators)
- **pH Control**: Auto-start NaOH pump when pH < LOW setpoint
- **Overcurrent Protection**: Auto-switch from Pump A to Pump B
- **Membrane Protection**: Auto-stop suction pump after 7 minutes (prevent membrane fouling)

### 5. Control Modes (4단계)

| 모드 | 조건 | Bi-LSTM | PSO | 송풍기 |
|------|------|---------|-----|--------|
| **정상 AI** | 수질 안정, 신뢰도 높음 | 5분 주기 | 30분 주기 | PSO 최적값 |
| **안전** | 이상 감지, 센서 불확실 | 10분 주기 | 중단 | DO 2.5 mg/L 고정 |
| **비상** | 수질 기준 초과 위험 | 중단 | 중단 | 최대 가동 |
| **Eco** | 저부하, 야간 | 30분 주기 | 1시간 주기 | 간헐 포기 |

## Working with This Repository

### Common Tasks

**Viewing Documentation:**
- All primary documents are in Markdown or HTML format
- Use Read tool for `.md` files
- HTML presentation can be viewed in browser or exported to PDF

**Understanding System Operations:**
1. Start with `operation_table.md` for control sequences
2. Review `project.md` sections 4-6 for technical specifications
3. Check `AI적용방법.md` for AI implementation details

**Modifying Documentation:**
- When editing tables in `operation_table.md`, maintain the ASCII table format with dashes (`-`) and pipes (`|`)
- Preserve Korean-English bilingual content
- Keep technical specifications aligned with actual operational data from the Gold Gang Engineering facility

### Key Technical Parameters

**Energy Targets:**
- Blower power consumption: 21.7 kW (50.4% of total facility power)
- Target energy reduction: 15%
- Annual savings potential: 30M KRW (for pilot facility)

**Water Quality Standards (Legal Requirements):**
- BOD: ≤ 10 mg/L (target: 8 mg/L, 20% safety margin)
- COD: ≤ 40 mg/L (target: 32 mg/L)
- SS: ≤ 10 mg/L (target: 8 mg/L)

**Control Intervals:**
- Data collection: 1-minute cycle
- Bi-LSTM prediction: 5-minute cycle
- PSO optimization: 30-minute cycle
- Sensor maintenance: Weekly (DO), Semi-weekly (pH)

### Implementation Phases

**Phase 1 (Months 1-4)**: Sensor network + data infrastructure (using existing + new COD/BOD sensors)
**Phase 2 (Months 5-8)**: AI model development + control algorithm + simulation
**Phase 3 (Months 9-12)**: Pilot testing + field demonstration + performance validation

**Testing Protocol**: Monitoring mode (2 weeks) → Semi-automatic (2 weeks) → Full automatic (4 weeks) → 72-hour unmanned operation

## Project Context

This is a **research planning document repository**, not an active codebase. The purpose is to:
1. Design and document the AI-based control system architecture
2. Plan the integration with existing PLC/SCADA infrastructure
3. Establish technical specifications based on real operational data
4. Prepare for R&D funding applications and stakeholder presentations

When working with this repository, focus on maintaining technical accuracy, preserving the connection to actual operational data from Gold Gang Engineering, and ensuring all proposed solutions comply with Korean water quality regulations (물환경보전법).

## Technical Terms Reference

- **DAF**: Dissolved Air Flotation (가압부상조)
- **MBR**: Membrane Bioreactor (분리막조)
- **EC-MBR**: Electrocoagulation + MBR (전기응집 결합형 MBR)
- **TMP**: Trans-Membrane Pressure (막간압력차)
- **DO**: Dissolved Oxygen (용존산소)
- **F/M Ratio**: Food-to-Microorganism ratio (부하율)
- **HRT**: Hydraulic Retention Time (체류시간)
- **SRT**: Solids Retention Time (고형물 체류시간)
- **TMS**: Tele-Monitoring System (원격모니터링 - 환경부 연동)
- **SCADA**: Supervisory Control and Data Acquisition
- **Feed-forward Control**: 사전 제어 (predictive/proactive control based on influent)
- **Feedback Control**: 사후 제어 (reactive control based on effluent)
- **MAPE**: Mean Absolute Percentage Error (예측 정확도 지표)
- **HEF**: Hailo Executable Format (Hailo NPU 모델 포맷)
- **OTA**: Over-The-Air (원격 업데이트)

## Document Navigation

문서 구조 및 네비게이션은 [README.md](README.md)를 참조하세요.

**주요 문서:**
- `README.md` - 프로젝트 개요, 3-Tier 아키텍처, 문서 네비게이션
- `PRD.md` - 소프트웨어 요구사항, 디렉토리 구조
- `TechArc.md` - 상세 기술 아키텍처 (4-Layer, Feed-forward 구현)
- `project.md` - R&D 제안서 (예산, 일정, KPI)


## 중요사항: 답변은 항상 한국어로 해줄것!