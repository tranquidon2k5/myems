# SPECIFICATION THIẾT KẾ KỸ THUẬT & HƯỚNG DẪN THỰC THI CHO CODING AGENT
## Hệ thống M&E & Năng lượng Thông minh trên DMP (MyEMS Analytics Layer)

> **Tài liệu tham chiếu:** 
> - [problem.md](file:///d:/VIN_AITC/myems/problem.md): Phát biểu bài toán, phạm vi, pipeline 5 bước và các module M1–M4.
> - [tele.md](file:///d:/VIN_AITC/myems/tele.md): Danh mục 132 telemetry keys + 16 attribute keys v1.13 trên DMP.

---

## 1. PHẠM VI & NGUYÊN TẮC HOẠT ĐỘNG CHO AGENT

### 1.1 Nguyên tắc bất biến (Invariable Rules)
1. **Read-Only / No Closed-Loop**: KHÔNG BAO GIỜ sinh code gửi lệnh điều khiển (`*_cmd`, `*_sp`) xuống phần cứng. Tất cả các key lệnh/setpoint chỉ được **ĐỌC** để làm ngữ cảnh so sánh.
2. **Resampling Grid**: Mọi xử lý time-series phải đưa về khung thời gian resample **15 phút chuẩn** (`00:00`, `00:15`, `00:30`, `00:45`).
3. **Fail-safe Quality**: Phân biệt rạch ròi trạng thái mất tín hiệu (`reachable = 0` hoặc missing row) với giá trị thực bằng 0.
4. **Type-safe & Schema Validation**: Mọi đầu ra (O1–O5) phải pass qua Pydantic Validation trước khi lưu DB hoặc trả về API.

### 1.2 Cấu trúc thư mục nguồn dự kiến (`src/`)
```
myems/
├── config/                  # Topology, mapping keys, threshold rules
├── src/
│   ├── models/              # Dataclasses & Pydantic models (O1 - O5)
│   ├── normalization/       # Resampler, Mapper, Quality Checker
│   ├── m1_power/            # Module M1: Electricity & Power Load
│   ├── m2_chiller/          # Module M2: Chiller Plant Performance & Optimization
│   ├── m3_water/            # Module M3: Water Pump Safety & Scheduling
│   ├── m4_air/              # Module M4: Indoor Air Quality & Ventilation
│   └── core/                # Recommendation Aggregator, Savings Evaluator & Reporter
├── tests/                   # Pytest suite cho từng module
├── problem.md
├── tele.md
└── spec.md
```

---

## 2. CHUẨN HOÁ INTERFACE ĐẦU RA (O1 - O5 CONTRACTS)

Mọi Coding Agent khi phát triển module phải tuân thủ đúng Pydantic Schema của các object đầu ra:

```python
# src/models/schemas.py
from pydantic import BaseModel, Field
from typing import Optional, Dict, Any, List
from datetime import datetime

class O1_Baseline(BaseModel):
    entity_id: str
    key: str
    ts: datetime
    expected: float
    lower: float
    upper: float
    baseline_layer: str  # "physics" | "history" | "peer"
    context: Dict[str, Any] = {}

class O2_AnomalyEvent(BaseModel):
    event_id: str
    module: str  # "M1_power" | "M2_chiller" | "M3_water" | "M4_air"
    entity_id: str
    type: str    # "dry_run" | "leakage" | "after_hours" | "low_cop" | "filter_clog"
    severity: str # "critical" | "warning" | "info"
    rule_or_model: str
    start: datetime
    end: Optional[datetime] = None
    evidence: Dict[str, Any]
    estimated_waste_kwh: float = 0.0
    suggested_action: str

class O3_ForecastPoint(BaseModel):
    ts: datetime
    p10: float
    p50: float
    p90: float

class O3_Forecast(BaseModel):
    entity_id: str
    key: str
    issued_at: datetime
    horizon: str  # "24h" | "1h" | "30m"
    step: str     # "15min"
    points: List[O3_ForecastPoint]

class O4_Recommendation(BaseModel):
    rec_id: str
    module: str
    valid_from: datetime
    valid_to: datetime
    action: str
    reason: str
    constraints_checked: List[str]
    est_saving_kwh: float = 0.0
    est_saving_vnd: float = 0.0
    status: str = "pending_operator"
```

---

## 3. DANH SÁCH AGENT TASK CARDS (PROMPT-READY TASKS)

Mỗi task dưới đây được thiết kế thành **Agent Task Card** riêng biệt, hoàn chỉnh thông tin để Coding Agent nhận và thực thi ngay lập tức.

---

### WORK PACKAGE 0: DATA PIPELINE & FOUNDATION

#### `TASK-0.1`: Khởi tạo Schema DB & Pydantic Models (O1–O5)
- **Target File**: `src/models/schemas.py`, `database/schema.sql`
- **Mục tiêu**: Định nghĩa toàn bộ Data Contracts Pydantic models và script tạo bảng PostgreSQL/TimescaleDB lưu trữ dữ liệu resample, O1, O2, O3, O4, O5.
- **Chi tiết từng bước**:
  1. Tạo `src/models/schemas.py` chứa các Class Pydantic: `O1_Baseline`, `O2_AnomalyEvent`, `O3_Forecast`, `O4_Recommendation`, `O5_Report`.
  2. Tạo `database/schema.sql` khởi tạo bảng `telemetry_clean_15m` (hypertable), `baselines`, `anomaly_events`, `forecasts`, `recommendations`.
  3. Thêm validation constraints (ví dụ: `p10 <= p50 <= p90`, `severity in ['critical', 'warning', 'info']`).
- **Lệnh Verification**: `pytest tests/test_schemas.py`
- **Mức độ ưu tiên**: High (Chặn toàn bộ dự án).

#### `TASK-0.2`: Ingestion, Resampling & Counter Reset Engine
- **Target File**: `src/normalization/resampler.py`
- **Mục tiêu**: Xây dựng module resample dữ liệu raw về chu kỳ 15m chuẩn và tính $\Delta E_{15m}$ từ counter tích lũy (`energy_active_kwh_total`, `water_volume_m3_total`).
- **Chi tiết từng bước**:
  1. Đọc Pandas Dataframe time-series bất kỳ có cột `ts` và các telemetry value.
  2. Resample time-series về lưới 15 phút (dùng `.resample('15min').mean()` cho biến trạng thái, `.last()` cho counter).
  3. Tính delta kỳ 15m cho biến counter: $\Delta = E(t) - E(t-15m)$.
  4. Nếu $\Delta < 0$ (do đồng hồ bị reset counter/thay mới), xử lý: gán $\Delta = E(t)$ hoặc dùng giá trị nội suy hợp lý.
- **Lệnh Verification**: `pytest tests/test_resampler.py`

#### `TASK-0.3`: Multi-source Key Mapping & Quality Checker
- **Target File**: `src/normalization/mapper.py`, `src/normalization/quality_checker.py`
- **Mục tiêu**: Mapping tên key từ Siemens NORIS, PHATTAI, ZKTeco về chuẩn DMP v1.13 và kiểm tra chất lượng mẫu (`GOOD`, `MISSING`, `INVALID`).
- **Chi tiết từng bước**:
  1. Xây dựng dictionary mapping giữa NORIS (`active_power_kw` $\rightarrow$ `power_active_kw`, `chw_supply_temp` $\rightarrow$ `chilled_water_supply_temp_c`).
  2. Tạo hàm `check_quality(df)`: Nếu `reachable == 0` gán flag `GATEWAY_DOWN`; nếu dữ liệu trống kéo dài gán `MISSING`.
- **Lệnh Verification**: `pytest tests/test_mapper_quality.py`

---

### WORK PACKAGE 1: MODULE M3 — NƯỚC & BƠM (AN TOÀN & LỊCH BƠM)

#### `TASK-1.1`: Rule Engine — Luật An Toàn Vật Lý Real-time (Lớp a)
- **Target File**: `src/m3_water/safety_rules.py`
- **Mục tiêu**: Bắt 6 kịch bản sự cố vật lý không cần dữ liệu lịch sử và sinh sự kiện `O2_AnomalyEvent`.
- **Logic 6 Luật**:
  1. **Chạy khô**: `pump_state == 1` AND (`tank_level_low == 1` OR `water_flow_lpm < 0.5`) duy trì $> 3$ phút $\rightarrow$ `Severity: critical`, type: `dry_run`.
  2. **Tràn bồn**: `tank_level_high == 1` AND `pump_state == 1` duy trì $> 5$ phút $\rightarrow$ `Severity: critical`, type: `overflow`.
  3. **Van kẹt**: `supply_valve_state_cmd == 1` nhưng `supply_valve_open_limit == 0` sau $30$s $\rightarrow$ `Severity: warning`, type: `valve_stuck`.
  4. **Lệnh không đáp ứng**: `pump_state_cmd != pump_state` sau $60$s $\rightarrow$ `Severity: warning`, type: `cmd_mismatch`.
  5. **Limit switch mâu thuẫn**: `open_limit == 1` AND `close_limit == 1` $\rightarrow$ `Severity: warning`, type: `switch_fault`.
  6. **Đóng cắt liên tục**: Số lần đổi `pump_state` trong 1 giờ $> 10$ lần $\rightarrow$ `Severity: warning`, type: `frequent_cycling`.
- **Lệnh Verification**: `pytest tests/test_m3_safety_rules.py`

#### `TASK-1.2`: Baseline Nước & Phát Hiện Rò Rỉ Ban Đêm
- **Target File**: `src/m3_water/leakage_detector.py`
- **Mục tiêu**: Tính chỉ số `night_min_flow` (lưu lượng nước nhỏ nhất trong khoảng 01:00 – 04:00 sáng). Nếu `night_min_flow > threshold` trong 3 ngày liên tiếp $\rightarrow$ Cảnh báo rò rỉ.
- **Lệnh Verification**: `pytest tests/test_leakage_detector.py`

#### `TASK-1.3`: Dự Báo Nhu Cầu Nước & Gợi Ý Lịch Bơm Theo Biểu Giá Điện (TOU)
- **Target File**: `src/m3_water/pump_scheduler.py`
- **Mục tiêu**: Lập lịch chạy bơm dồn vào giờ thấp điểm (22:00 – 04:00) dựa trên biểu giá điện.
- **Ràng buộc cứng**: Mực nước bồn không bao giờ vi phạm `tank_level_low = 1`, tôn trọng `min_runtime` và `min_offtime`.
- **Đầu ra**: Danh sách `O4_Recommendation` kèm ước tính $VND$ tiết kiệm.
- **Lệnh Verification**: `pytest tests/test_pump_scheduler.py`

---

### WORK PACKAGE 2: MODULE M1 — ĐIỆN NĂNG (ANCHOR DỮ LIỆU)

#### `TASK-2.1`: Calculator Đại Lượng Điện Năng Dẫn Xuất
- **Target File**: `src/m1_power/derived_metrics.py`
- **Mục tiêu**: Tính toán các biến dẫn xuất từ 13 key Power Meter:
  - $kWh_{15m} = \Delta energy\_active\_kwh\_total$
  - `phase_voltage_unbalance_pct` = $\frac{\max|V_{Lx} - V_{avg}|}{V_{avg}} \times 100\%$
  - `phase_current_unbalance_pct` = $\frac{\max|I_{Lx} - I_{avg}|}{I_{avg}} \times 100\%$
  - `after_hours_kwh`: kWh tiêu thụ ngoài khung giờ vận hành (theo Calendar).
- **Lệnh Verification**: `pytest tests/test_m1_derived.py`

#### `TASK-2.2`: Baseline Phụ Tải Điện 3 Lớp
- **Target File**: `src/m1_power/baseline_engine.py`
- **Mục tiêu**: Xây dựng mức kỳ vọng `expected`, `lower`, `upper` cho `power_active_kw` (15m grid):
  - Lớp a (Vật lý): $0 \le P \le P_{rated}$.
  - Lớp b (Lịch sử): Trung bình tĩnh theo [Loại ngày (Workday/Weekend) × Khung giờ 15m × Dải nhiệt độ ngoài trời].
  - Lớp c (Peer): So sánh công suất trung bình với các tủ điện cùng loại.
- **Đầu ra**: Object `O1_Baseline`.
- **Lệnh Verification**: `pytest tests/test_m1_baseline.py`

#### `TASK-2.3`: Mô Hình Dự Báo Phụ Tải Điện 24h (Load Forecaster)
- **Target File**: `src/m1_power/load_forecaster.py`
- **Mục tiêu**: Train & predict phụ tải điện `power_active_kw` cho 24h tới (96 steps × 15m) với dải tin cậy P10, P50, P90.
- **Input Features**: Lịch làm việc, Nhiệt độ/Độ ẩm dự báo (PHATTAI), Phụ tải 7 ngày gần nhất.
- **Đầu ra**: Object `O3_Forecast`. (MAPE cam kết $\le 15\%$).
- **Lệnh Verification**: `pytest tests/test_m1_forecaster.py`

#### `TASK-2.4`: Detector Bất Thường Điện Năng & Lãng Phí
- **Target File**: `src/m1_power/anomaly_detector.py`
- **Mục tiêu**: Phát hiện:
  1. *Chạy ngoài giờ*: Công suất ngoài giờ $> 1.5 \times Baseline_{after\_hours}$.
  2. *Hệ số công suất thấp*: $power\_factor < 0.85$ kéo dài $> 30$ phút.
  3. *Lệch pha*: Voltage/Current unbalance $> 5\%$.
- **Lệnh Verification**: `pytest tests/test_m1_anomaly.py`

---

### WORK PACKAGE 3: MODULE M2 — CHILLER PLANT (VERTICAL CHỦ ĐẠO)

#### `TASK-3.1`: Calculator KPI Hiệu Suất Chiller & Đối Soát Nhiệt
- **Target File**: `src/m2_chiller/metrics_calculator.py`
- **Mục tiêu**:
  - $\Delta T_{chw} = chilled\_water\_return\_temp\_c - chilled\_water\_supply\_temp\_c$
  - $COP = \frac{cooling\_kw}{active\_power\_kw}$
  - $kW/kW_{cooling} = \frac{active\_power\_kw}{cooling\_kw}$
  - $Q_{thermal\_check} = 4.186 \times \frac{water\_flow\_lpm}{60} \times \Delta T_{chw}$ (đối soát với `cooling_kw`).
- **Lệnh Verification**: `pytest tests/test_m2_metrics.py`

#### `TASK-3.2`: Mô Hình Dự Báo Tải Lạnh 24h (Cooling Load Forecaster)
- **Target File**: `src/m2_chiller/cooling_forecaster.py`
- **Mục tiêu**: Dự báo nhu cầu `cooling_kw` cho 24h tới (96 steps 15m) với các khoảng P10, P50, P90.
- **Input Features**: Temp, Humidity, Calendar, ZKTeco Occupancy Proxy.
- **Đầu ra**: Object `O3_Forecast` (MAPE cam kết $\le 15\%$).
- **Lệnh Verification**: `pytest tests/test_m2_cooling_forecast.py`

#### `TASK-3.3`: Performance Curve & Suy Giảm Hiệu Suất Chiller
- **Target File**: `src/m2_chiller/performance_curve.py`
- **Mục tiêu**: 
  - Xây dựng đường cong hiệu suất $kW/kW_{cooling}$ theo $\% \text{Tải } (chiller\_load\_pct)$ và Nhiệt độ nước làm mát / ngoài trời.
  - Cảnh báo khi $kW/kW_{cooling}$ thực tế cao hơn $> 10\%$ so với đường cong chuẩn của chính máy đó.
  - Cảnh báo *Low $\Delta T$ syndrome* khi $\Delta T_{chw} < 2.0^\circ C$ kéo dài khi máy đang chạy.
- **Lệnh Verification**: `pytest tests/test_m2_performance_curve.py`

#### `TASK-3.4`: Optimization Engine — Tổ Hợp Chiller & Setpoint $T_{supply\_sp}$
- **Target File**: `src/m2_chiller/plant_optimizer.py`
- **Mục tiêu**: Đề xuất tổ hợp Chiller chạy (ví dụ: Chạy CH-01 + CH-03 thay vì CH-01 + CH-02) và $T_{supply\_sp}$ tối ưu cho từng khung giờ 24h tới.
- **Ràng buộc**:
  - Nhiệt độ nước lạnh cấp $T_{supply} \le 7.5^\circ C$.
  - Tôn trọng `runtime_min` (không bật/tắt liên tục trong $< 30$ phút).
  - Ước tính $kWh$ và $VND$ tiết kiệm.
- **Đầu ra**: Object `O4_Recommendation`.
- **Lệnh Verification**: `pytest tests/test_m2_plant_optimizer.py`

---

### WORK PACKAGE 4: MODULE M4 — KHÔNG KHÍ & THÔNG GIÓ

#### `TASK-4.1`: Spatial Topology & Occupancy Proxy Adapter
- **Target File**: `src/m4_air/occupancy_adapter.py`, `config/topology_m4.json`
- **Mục tiêu**: Mapping Cửa ZKTeco $\leftrightarrow$ Vùng $CO_2$ $\leftrightarrow$ PAU/quạt. Tính `occupancy_proxy` = $\Delta record\_count$ trong kỳ 15m.
- **Lệnh Verification**: `pytest tests/test_m4_occupancy.py`

#### `TASK-4.2`: Dự Báo $CO_2$ Vùng & Bụi Mịn $PM_{2.5}$
- **Target File**: `src/m4_air/air_forecaster.py`
- **Mục tiêu**: Dự báo $CO_2$ trong 15–30 phút tới theo mô hình cân bằng khối lượng và dự báo $PM_{2.5}$ ngoài trời từ PHATTAI.
- **Lệnh Verification**: `pytest tests/test_m4_air_forecast.py`

#### `TASK-4.3`: Anomaly Detector — Tắc Lọc Wind Side & Van Gió Kẹt
- **Target File**: `src/m4_air/anomaly_detector.py`
- **Mục tiêu**: Theo dõi xu hướng tăng của chênh áp qua lọc `diff_pressure_pa` tại cùng cấp `fan_speed_pct` để cảnh báo tắc lọc gió.
- **Lệnh Verification**: `pytest tests/test_m4_air_anomaly.py`

#### `TASK-4.4`: Ventilation Optimizer — Tiết Kiệm Gió Tươi & Quạt
- **Target File**: `src/m4_air/ventilation_optimizer.py`
- **Mục tiêu**: Đề xuất giảm tốc độ quạt khi zone vắng người; giảm tỷ lệ gió tươi tạm thời khi $PM_{2.5}$ ngoài trời quá cao nhưng $CO_2$ còn trong ngưỡng an toàn.
- **Đầu ra**: Object `O4_Recommendation`.
- **Lệnh Verification**: `pytest tests/test_m4_ventilation_opt.py`

---

### WORK PACKAGE 5: CORE FRAMEWORK & DELIVERY SERVICES

#### `TASK-5.1`: Core Recommendation Engine & Conflict Resolver
- **Target File**: `src/core/recommendation_engine.py`
- **Mục tiêu**: Tổng hợp khuyến nghị từ M1, M2, M3, M4; loại bỏ các gợi ý mâu thuẫn (vd: M2 đòi tăng lưu lượng nhưng M3 đòi giảm bơm); tính tổng $VND$ tiết kiệm theo khung giá điện TOU.
- **Lệnh Verification**: `pytest tests/test_core_rec_engine.py`

#### `TASK-5.2`: Reporting & Backtest Engine (O5 Service)
- **Target File**: `src/core/reporting_service.py`, `src/core/backtester.py`
- **Mục tiêu**: Sinh báo cáo O5 theo ngày/tuần/tháng; chạy backtest đánh giá giả định tiết kiệm trên dữ liệu lịch sử.
- **Lệnh Verification**: `pytest tests/test_core_reporting.py`

#### `TASK-5.3`: FastAPI REST Endpoints & Data Delivery
- **Target File**: `src/api/main.py`, `src/api/routers/analytics.py`
- **Mục tiêu**: Expose các API REST endpoints phục vụ Dashboard UI MyEMS đọc dữ liệu O1, O2, O3, O4, O5.
- **Lệnh Verification**: `pytest tests/test_api_endpoints.py`

---

## 4. QUY TRÌNH KHI AGENT NHẬN TASK (AGENT WORKFLOW)

Khi một Agent bắt đầu thực thi 1 task bất kỳ (ví dụ `TASK-3.1`):
1. **Đọc kĩ Task Card** tương ứng trong `spec.md` (Target file, Input/Output schemas, Verification commands).
2. **Kiểm tra File Schema**: Đảm bảo các Pydantic class từ `src/models/schemas.py` được import đúng.
3. **Viết Code Chức Năng** vào đúng `Target File`.
4. **Viết Unit Test Tương Ứng** trong thư mục `tests/`.
5. **Chạy Lệnh Verification**: Chạy `pytest tests/...` để tự xác minh kết quả thành công trước khi hoàn thành task.
