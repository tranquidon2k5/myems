# Bài toán A — Năng lượng & M&E thông minh trên DMP

**Baseline → Phát hiện bất thường → Dự báo → Gợi ý tối ưu**

Gom đề tài 1 (Điện), 3 (Chiller) làm trục chính; mở rộng cùng khung cho đề tài 2 (Nước), 6 (AQI) và 7 (CO2).
Nguồn key: *DMP telemetry keys v1.13 (132 telemetry + 16 attribute)*.

---

## 1. Phát biểu bài toán

Hệ M&E của tòa nhà (điện, chiller, nước, thông gió) đang vận hành theo lịch cố định và logic ngưỡng đơn giản. Hệ thống **không biết mức vận hành "bình thường" của từng thiết bị/khu vực là bao nhiêu**, nên:

1. Lãng phí âm thầm (thiết bị chạy ngoài giờ, chạy non tải, hiệu suất suy giảm dần) không ai nhìn thấy.
2. Sự cố (rò rỉ nước, bơm chạy khô, chiller mất hiệu suất, lọc tắc) chỉ lộ ra khi đã hỏng hoặc đã gây hậu quả.
3. Không tính trước được phụ tải, nên không khai thác được chênh lệch giá điện giữa các khung giờ (~3 lần giữa cao điểm và thấp điểm).

**Bài toán:** Xây dựng một lớp phân tích dùng chung trên DMP, nhận telemetry sẵn có của các hệ M&E, và với mỗi thiết bị / khu vực:

- học **baseline** (mức vận hành kỳ vọng theo giờ, loại ngày, thời tiết, lượng người);
- **phát hiện bất thường** khi giá trị thực lệch khỏi baseline hoặc vi phạm luật vật lý;
- **dự báo** phụ tải / nhu cầu trong 1–24h tới;
- đưa ra **gợi ý vận hành** (bật/tắt, số máy chạy, lịch bơm, setpoint) kèm ước tính kWh và tiền điện tiết kiệm được.

Người vận hành là người ra quyết định cuối cùng. Hệ thống **không ghi lệnh xuống thiết bị**.

---

## 2. Phạm vi

### 2.1 Trong phạm vi

| Hạng mục | Mô tả |
|---|---|
| Framework dùng chung | Pipeline 5 bước (mục 3) chạy được cho mọi miền, cấu hình theo module |
| M1 — Điện năng | Anchor dữ liệu. Baseline tiêu thụ, phát hiện lãng phí, dự báo phụ tải tổng |
| M2 — Chiller | Vertical chủ đạo về giá trị. Dự báo tải lạnh, mô hình hiệu suất từng máy, gợi ý điểm vận hành |
| M3 — Nước / bơm | Cảnh báo an toàn theo luật, phát hiện rò rỉ và suy giảm bơm, gợi ý lịch bơm theo giá điện |
| M4 — Không khí / CO2 | Dự báo CO2/PM2.5 ngắn hạn, gợi ý tăng/giảm gió tươi |
| Đầu ra | Sự kiện bất thường, chuỗi dự báo, gợi ý vận hành, báo cáo tiết kiệm (mục 5) |

### 2.2 Ngoài phạm vi (bắt buộc)

- **Không closed-loop**: không ghi các key `*_cmd`, `*_sp` xuống thiết bị. Các key này chỉ được **đọc** để biết hệ đang được ra lệnh gì.
- Loại bỏ GĐ3 "tự động điều khiển" của đề tài 6 và 7.
- Không thay thế logic an toàn cục bộ của PLC/BMS (phao bồn, interlock bơm, bảo vệ chiller).
- Không lắp thêm cảm biến trong phạm vi thực tập; chỉ ghi nhận thành đề xuất nếu thiếu dữ liệu.

---

## 3. Kiến trúc pipeline dùng chung

```
Telemetry DMP ──┐
Attribute DMP ──┼─► [1] Chuẩn hóa ─► [2] Baseline ─► [3] Bất thường ─► [4] Dự báo ─► [5] Gợi ý
Dữ liệu ngoài ──┘         │               │                 │                │             │
                          ▼               ▼                 ▼                ▼             ▼
                  Chuỗi sạch 15'   Mức kỳ vọng +      Sự kiện bất       Chuỗi dự báo   Khuyến nghị +
                  + cờ chất lượng  dải tin cậy        thường có          + khoảng tin   ước tính tiết
                                                      bằng chứng         cậy            kiệm
```

| Bước | Việc làm | Ghi chú |
|---|---|---|
| 1. Chuẩn hóa | Resample về lưới 15 phút; chuyển counter tích lũy (`*_total`) thành lượng tiêu thụ theo kỳ, xử lý reset counter; đồng nhất đơn vị và tên key giữa DMP chuẩn và NORIS; gắn cờ mất dữ liệu | Phân biệt "mất dữ liệu" với "giá trị bằng 0" |
| 2. Baseline — 3 lớp | **(a) Luật vật lý**: ràng buộc không cần lịch sử. **(b) Lịch sử chính thiết bị**: profile theo loại ngày × khung giờ, có điều kiện theo nhiệt độ ngoài trời, lượng người. **(c) So ngang tập thiết bị**: so với các thiết bị cùng loại, cùng model | Lấy từ phương pháp đề tài 2 |
| 3. Bất thường | Điểm (spike), đoạn (drift, chạy ngoài giờ), trạng thái (cmd ≠ phản hồi), hiệu suất (kW/kW lạnh tăng dần) | Mỗi sự kiện kèm key, khoảng thời gian, baseline, độ lệch |
| 4. Dự báo | Phụ tải điện, tải lạnh, nhu cầu nước, CO2/PM2.5 | Horizon theo module |
| 5. Gợi ý | Tối ưu có ràng buộc: an toàn → tiện nghi → chi phí điện | Chỉ đề xuất, người vận hành xác nhận |

---

## 4. Input tổng quát

### 4.1 Telemetry DMP (time-series)

Chi tiết theo module ở mục 6; ma trận tổng hợp ở mục 7.

### 4.2 Attribute DMP (metadata)

| Key | Dùng để |
|---|---|
| `device_type` | Định tuyến thiết bị vào module |
| `device_model` | Gom nhóm cho baseline lớp (c) — so ngang cùng model |
| `device_label` | Hiển thị trong cảnh báo / gợi ý |
| `vendor_device_id`, `serial_number` | Đối soát thiết bị giữa DMP và NORIS |
| `firmware_version` | Loại trừ bất thường giả do đổi firmware |

### 4.3 Dữ liệu ngoài telemetry (cần thu thập / cấu hình)

| Dữ liệu | Dạng | Bắt buộc | Nguồn |
|---|---|---|---|
| Biểu giá điện theo khung giờ (thấp điểm / bình thường / cao điểm) | Bảng cấu hình giờ + đơn giá | ✔ | Hợp đồng điện của tòa nhà |
| Lịch tòa nhà: ngày làm việc, cuối tuần, ngày lễ, giờ vận hành | Calendar | ✔ | Ban quản lý tòa nhà |
| Topology: thiết bị → đồng hồ điện → khu vực / tầng | Bảng quan hệ | ✔ | SMCP / hồ sơ M&E |
| Thông số định mức: công suất lạnh chiller, công suất bơm, đường cong bơm, dung tích bồn | Bảng tĩnh | ✔ (M2, M3) | Hồ sơ thiết bị |
| Mapping cửa ZKTeco ↔ vùng cảm biến CO2 ↔ PAU/quạt phục vụ | Bảng quan hệ | ✔ (M4) | Khảo sát hiện trường |
| Dự báo thời tiết 24h (nhiệt độ, độ ẩm) | API | Tùy chọn | Dịch vụ thời tiết; nếu không có thì dùng `temperature`, `humidity` của PHATTAI làm lịch sử |
| Nhật ký bảo trì (thay lọc, vệ sinh dàn, sửa bơm) | Log | Tùy chọn | Phiếu bảo trì — dùng làm nhãn đánh giá |

---

## 5. Output tổng quát

### 5.1 O1 — Baseline

Chuỗi mức kỳ vọng và dải tin cậy cho từng (thiết bị hoặc khu vực, key), lưới 15 phút.

```json
{
  "entity_id": "PM-B1-MAIN",
  "key": "power_active_kw",
  "ts": "2026-09-16T14:00:00+07:00",
  "expected": 412.5,
  "lower": 380.1,
  "upper": 446.0,
  "baseline_layer": "history",
  "context": {"day_type": "workday", "outdoor_temp_c": 33.1}
}
```

### 5.2 O2 — Sự kiện bất thường

```json
{
  "event_id": "A-20260916-0042",
  "module": "M3_water",
  "entity_id": "PUMP-ROOF-02",
  "type": "dry_run",
  "severity": "critical",
  "rule_or_model": "physics_rule",
  "start": "2026-09-16T02:15:00+07:00",
  "end": null,
  "evidence": {
    "pump_state": 1,
    "tank_level_low": 0,
    "water_flow_lpm": 0.4,
    "pressure_supply_bar": 0.1
  },
  "estimated_waste_kwh": 3.2,
  "suggested_action": "Kiểm tra bồn hút và dừng bơm PUMP-ROOF-02"
}
```

Loại `severity`: `critical` (an toàn, báo ngay, không cần baseline) · `warning` (lệch baseline có ý nghĩa) · `info` (xu hướng suy giảm).

### 5.3 O3 — Dự báo

```json
{
  "entity_id": "CHILLER-PLANT",
  "key": "cooling_kw",
  "issued_at": "2026-09-16T00:00:00+07:00",
  "horizon": "24h",
  "step": "15min",
  "points": [{"ts": "...", "p50": 1850, "p10": 1700, "p90": 2010}]
}
```

### 5.4 O4 — Gợi ý vận hành

```json
{
  "rec_id": "R-20260916-007",
  "module": "M2_chiller",
  "valid_from": "2026-09-16T09:00:00+07:00",
  "valid_to": "2026-09-16T11:30:00+07:00",
  "action": "Chạy CH-01 + CH-03 thay vì CH-01 + CH-02",
  "reason": "CH-02 có kW/kW lạnh cao hơn 14% so với baseline cùng tải",
  "constraints_checked": ["chw_supply_temp <= 7.5C", "min_runtime", "comfort"],
  "est_saving_kwh": 96,
  "est_saving_vnd": 250000,
  "status": "pending_operator"
}
```

### 5.5 O5 — Báo cáo định kỳ (ngày / tuần / tháng)

- Tiêu thụ thực tế vs baseline theo hệ, theo khu vực.
- Top thiết bị lãng phí / suy giảm hiệu suất.
- Số gợi ý đã đưa ra, số gợi ý được chấp nhận, tiết kiệm ước tính.
- Chất lượng dữ liệu: tỷ lệ mất mẫu theo thiết bị.

---

## 6. Chi tiết theo module

Quy ước cột **Vai trò**: `Đích` = biến cần baseline / dự báo · `Giải thích` = biến ngữ cảnh đưa vào mô hình · `Trạng thái` = dùng cho luật và lọc · `Đọc lệnh` = key `_cmd`/`_sp`, chỉ đọc.

### 6.1 M1 — Điện năng (đề tài 1 · anchor dữ liệu)

**Câu hỏi cần trả lời**
- Tòa nhà / từng tủ / từng khu đang tiêu thụ nhiều hơn mức kỳ vọng bao nhiêu, vào lúc nào?
- Phụ tải 24h tới là bao nhiêu, bao nhiêu rơi vào giờ cao điểm?
- Có dấu hiệu bất thường về chất lượng điện (lệch pha, PF thấp) không?

**Telemetry sử dụng — nhóm Power meter (13/13)**

| Key | Đơn vị | Vai trò | Dùng cho |
|---|---|---|---|
| `energy_active_kwh_total` | kWh | Đích | Counter → kWh/15 phút; baseline tiêu thụ, báo cáo tiền điện |
| `power_active_kw` | kW | Đích | Baseline & dự báo phụ tải, phát hiện spike / chạy ngoài giờ |
| `energy_reactive_kvarh_total` | kvarh | Trạng thái | Theo dõi công suất phản kháng |
| `power_factor` | – | Trạng thái | Cảnh báo PF thấp (nguy cơ phạt tiền phản kháng) |
| `voltage_v`, `voltage_l1_v`, `voltage_l2_v`, `voltage_l3_v` | V | Trạng thái | Luật: lệch áp giữa các pha, sụt áp |
| `current_a`, `current_l1_a`, `current_l2_a`, `current_l3_a` | A | Trạng thái | Luật: mất cân bằng dòng pha |
| `frequency_hz` | Hz | Trạng thái | Luật: tần số ngoài dải (thường khi chạy máy phát) |

**Telemetry bổ sung**

| Key | Nhóm | Vai trò | Dùng cho |
|---|---|---|---|
| `active_energy_kwh`, `active_power_kw`, `voltage_l1`, `voltage_l2`, `voltage_l3` | §2.1 NORIS | Đích / Trạng thái | Nguồn thay thế khi đồng hồ chưa đẩy real-time vào DMP; cần mapping sang key chuẩn |
| `temperature`, `humidity` | §2.2 PHATTAI | Giải thích | Điều kiện hóa baseline theo thời tiết |
| `lighting_knx_state` | Dùng chung | Giải thích | Giải thích bước nhảy tải chiếu sáng, phát hiện đèn bật ngoài giờ |
| `state_on_off` | Dùng chung | Giải thích | Trạng thái tải lớn trên cùng tủ |
| `record_count` | ZKTeco | Giải thích | Δ theo 15 phút ≈ lưu lượng người ra vào (xem lưu ý mục 8) |

**Đại lượng dẫn xuất**
- `kwh_15m = Δ energy_active_kwh_total` (bỏ Δ âm do reset counter).
- `phase_voltage_unbalance_pct = max|V_Lx − V_avg| / V_avg × 100`.
- `phase_current_unbalance_pct` tương tự cho dòng.
- `peak_share_pct` = kWh giờ cao điểm / tổng kWh ngày.
- `after_hours_kwh` = kWh ngoài giờ vận hành theo calendar.

**Output cụ thể**
- Baseline `power_active_kw` và `kwh_15m` theo đồng hồ / khu vực.
- Bất thường: chạy ngoài giờ, tải nền (base load) tăng dần, spike, PF thấp, mất cân bằng pha > ngưỡng.
- Dự báo phụ tải 24h, bước 15 phút, kèm phân rã theo khung giá.
- Báo cáo: kWh và tiền điện thực tế vs baseline.

### 6.2 M2 — Chiller (đề tài 3 · vertical chủ đạo)

**Câu hỏi cần trả lời**
- Tải lạnh 24h tới là bao nhiêu?
- Mỗi chiller đang chạy với hiệu suất bao nhiêu so với chính nó trước đây và so với máy cùng loại?
- Với tải dự báo, nên chạy máy nào, bao nhiêu máy, khởi động lúc mấy giờ, nhiệt độ nước lạnh cấp bao nhiêu?

**Telemetry sử dụng — nhóm HVAC · chiller (9/9)**

| Key | Đơn vị | Vai trò | Dùng cho |
|---|---|---|---|
| `cooling_kw` | kW | Đích | Tải lạnh — biến dự báo chính |
| `chiller_load_pct` | % | Đích | Đường cong hiệu suất theo % tải |
| `chilled_water_supply_temp_c` | °C | Trạng thái | Ràng buộc tiện nghi; biến gợi ý setpoint |
| `chilled_water_return_temp_c` | °C | Trạng thái | ΔT nước lạnh; phát hiện "low ΔT syndrome" |
| `chilled_water_supply_pressure_bar` | bar | Trạng thái | Luật áp suất, suy giảm lưu lượng |
| `chilled_water_return_pressure_bar` | bar | Trạng thái | Chênh áp cấp–hồi |
| `chiller_state` | – | Trạng thái | Máy đang chạy / dừng / lỗi |
| `chiller_state_cmd` | – | Đọc lệnh | So lệnh vs trạng thái thực |
| `runtime_min` | min | Trạng thái | Cân bằng giờ chạy giữa các máy, min runtime |

**Telemetry sử dụng — §2.1 Siemens NORIS (13/13)**

| Key | Đơn vị | Map sang key chuẩn / vai trò |
|---|---|---|
| `active_power_kw` | kW | Công suất điện của chiller — **bắt buộc** để tính hiệu suất |
| `active_energy_kwh` | kWh | Điện năng tích lũy của chiller |
| `cooling_load_kw` | kW | ≈ `cooling_kw` |
| `chw_supply_temp` | °C | ≈ `chilled_water_supply_temp_c` |
| `chw_return_temp` | °C | ≈ `chilled_water_return_temp_c` |
| `evaporator_pressure` | bar | Trạng thái — suy giảm dàn bay hơi, thiếu gas |
| `compressor_status` | – | Trạng thái máy nén |
| `op_mode` | – | Chế độ vận hành (loại dữ liệu chạy test / bảo trì) |
| `run_hours` | h | ≈ `runtime_min` |
| `voltage_l1`, `voltage_l2`, `voltage_l3` | V | Trạng thái điện cấp cho chiller |
| `water_volume_m3` | m³ | Nước bù (tháp giải nhiệt) nếu gắn với plant |

**Telemetry bổ sung**

| Key | Nhóm | Vai trò | Dùng cho |
|---|---|---|---|
| `power_active_kw`, `energy_active_kwh_total` | Power meter | Đích | Nếu chiller có đồng hồ riêng trong DMP (thay NORIS) |
| `cooling_tower_fan_state` | Air side | Trạng thái | Tính công suất cả plant (chiller + tháp) |
| `pump_state`, `pump_speed_pct`, `water_flow_lpm` | Nước | Trạng thái | Bơm nước lạnh / nước giải nhiệt thuộc plant; lưu lượng để kiểm chéo tải lạnh |
| `supply_air_temp_c`, `fan_state` | Air side | Giải thích | Nhu cầu lạnh phía AHU/PAU |
| `temperature`, `humidity` | PHATTAI | Giải thích | Biến thời tiết chính cho dự báo tải lạnh |
| `record_count` | ZKTeco | Giải thích | Proxy lượng người trong tòa nhà |
| `fault`, `mode` | Dùng chung | Trạng thái | Loại dữ liệu khi lỗi / chế độ đặc biệt |

**Đại lượng dẫn xuất**
- `chw_delta_t = chilled_water_return_temp_c − chilled_water_supply_temp_c`.
- `kw_per_kw_cooling = active_power_kw / cooling_kw` (càng thấp càng tốt); `COP = cooling_kw / active_power_kw`.
- `kw_per_tr = active_power_kw / (cooling_kw / 3.517)` nếu cần đơn vị RT.
- `plant_kw` = chiller + bơm + quạt tháp (khi đủ dữ liệu).
- Kiểm chéo: `Q ≈ 4.186 × (water_flow_lpm / 60) × chw_delta_t` (kW, nếu có flow trên vòng nước lạnh).

**Output cụ thể**
- Dự báo `cooling_kw` 24h, bước 15 phút (P10/P50/P90).
- Đường cong hiệu suất `kw_per_kw_cooling` theo `chiller_load_pct` và nhiệt độ ngoài trời cho từng máy.
- Bất thường: hiệu suất suy giảm so với đường cong của chính máy; ΔT thấp kéo dài; `chiller_state_cmd` ≠ `chiller_state`; chạy khi tải rất thấp.
- Gợi ý: tổ hợp máy chạy theo từng khung giờ, thời điểm khởi động buổi sáng, setpoint nước lạnh cấp trong dải cho phép, kèm kWh / tiền điện tiết kiệm ước tính.

### 6.3 M3 — Nước / bơm / van / bồn (đề tài 2)

**Câu hỏi cần trả lời**
- Có bơm nào đang chạy khô, van nào kẹt, bồn nào sắp cạn / tràn không? *(an toàn — báo ngay)*
- Có rò rỉ không (tiêu thụ ban đêm cao bất thường)?
- Bơm nào đang suy giảm (cùng tốc độ nhưng áp / lưu lượng giảm)?
- Có thể dời giờ bơm lên bồn sang thấp điểm mà vẫn đảm bảo mực nước an toàn không?

**Telemetry sử dụng — nhóm Nước / bơm / van / bồn (20/20)**

| Key | Đơn vị | Vai trò | Dùng cho |
|---|---|---|---|
| `water_volume_m3_total` | m³ | Đích | Counter → m³/15 phút; baseline tiêu thụ, rò rỉ ban đêm |
| `water_flow_lpm` | L/min | Đích | Baseline lưu lượng, dự báo nhu cầu, luật chạy khô |
| `pump_state` | – | Trạng thái | Luật chạy khô, đếm số lần khởi động |
| `pump_state_cmd` | – | Đọc lệnh | Lệnh vs trạng thái thực |
| `pump_speed_pct` | % | Trạng thái | Đường cong bơm (tốc độ → áp, lưu lượng) |
| `pump_speed_pct_sp` | % | Đọc lệnh | Setpoint vs thực tế |
| `pressure_supply_bar` | bar | Trạng thái | Suy giảm bơm, rò rỉ, tắc |
| `pressure_return_bar` | bar | Trạng thái | Chênh áp hệ |
| `water_supply_temp_c` | °C | Trạng thái | Luật nhiệt (bơm nóng khi chạy khô / tuần hoàn kín) |
| `water_return_temp_c` | °C | Trạng thái | ΔT vòng tuần hoàn |
| `tank_level_low` | bool | Trạng thái | Ràng buộc an toàn, luật chạy khô |
| `tank_level_medium` | bool | Trạng thái | Ước lượng mực bồn rời rạc 3 mức |
| `tank_level_high` | bool | Trạng thái | Luật tràn bồn |
| `supply_valve_open_limit`, `supply_valve_close_limit` | bool | Trạng thái | Phản hồi vị trí van cấp |
| `supply_valve_state_cmd` | – | Đọc lệnh | Luật van kẹt |
| `return_valve_open_limit`, `return_valve_close_limit` | bool | Trạng thái | Phản hồi vị trí van hồi |
| `return_valve_state_cmd` | – | Đọc lệnh | Luật van kẹt |
| `bypass_valve_state_cmd` | – | Đọc lệnh | Giải thích thay đổi áp khi mở bypass |

**Telemetry bổ sung**

| Key | Nhóm | Vai trò | Dùng cho |
|---|---|---|---|
| `power_active_kw`, `energy_active_kwh_total` | Power meter | Đích | Điện năng bơm → chi phí, hiệu suất kWh/m³ |
| `fault`, `mode`, `state_on_off` | Dùng chung | Trạng thái | Lọc dữ liệu lỗi / chế độ tay |
| `record_count` | ZKTeco | Giải thích | Proxy nhu cầu nước theo lượng người |

**Luật an toàn (lớp a — không cần baseline, triển khai trước)**

| Luật | Điều kiện (gợi ý, cần hiệu chỉnh) | Mức |
|---|---|---|
| Chạy khô | `pump_state` = ON và (`tank_level_low` = báo cạn **hoặc** `water_flow_lpm` ≈ 0) kéo dài > N phút | critical |
| Tràn bồn | `tank_level_high` = báo đầy và bơm lên bồn vẫn ON | critical |
| Van kẹt | `*_valve_state_cmd` = OPEN nhưng `*_valve_open_limit` không lên sau T giây (tương tự CLOSE) | warning |
| Lệnh không đáp ứng | `pump_state_cmd` ≠ `pump_state` sau T giây | warning |
| Limit switch mâu thuẫn | `*_open_limit` và `*_close_limit` cùng active | warning |
| Bơm đóng cắt liên tục | Số lần `pump_state` đổi trạng thái / giờ > ngưỡng | warning |

> Cần xác nhận với hiện trường ngữ nghĩa của `tank_level_*` (tiếp điểm thường đóng hay thường mở) trước khi bật luật.

**Đại lượng dẫn xuất**
- `m3_15m = Δ water_volume_m3_total`.
- `night_min_flow` = lưu lượng tối thiểu 1h–4h sáng → chỉ báo rò rỉ.
- `pump_head_proxy = pressure_supply_bar − pressure_return_bar`, so với `pump_speed_pct` và `water_flow_lpm` → suy giảm bơm.
- `kwh_per_m3` = điện bơm / m³ bơm.
- `tank_fill_time` = thời gian từ `tank_level_low` đến `tank_level_high` → ước lượng tốc độ tiêu thụ.

**Output cụ thể**
- Sự kiện an toàn real-time (bảng trên).
- Baseline tiêu thụ nước, bất thường rò rỉ ban đêm, suy giảm bơm.
- Dự báo nhu cầu nước theo giờ trong ngày.
- Gợi ý lịch bơm lên bồn: dồn vào giờ thấp điểm, **ràng buộc cứng**: mực bồn không bao giờ xuống `tank_level_low` trong dự báo, tôn trọng min on/off time của bơm.

### 6.4 M4 — Không khí / CO2 / thông gió (đề tài 6 + 7)

**Câu hỏi cần trả lời**
- CO2 / PM2.5 trong 15–60 phút tới có vượt ngưỡng không?
- Khu vực nào đang cấp gió thừa khi vắng người (lãng phí quạt / lạnh)?
- Khi ngoài trời ô nhiễm, có nên giảm gió tươi tạm thời không?
- Lọc gió có đang tắc dần không?

**Telemetry sử dụng — nhóm HVAC · air side (PAU / quạt / tháp) (16/16)**

| Key | Đơn vị | Vai trò | Dùng cho |
|---|---|---|---|
| `co2_ppm` | ppm | Đích | Dự báo CO2 trong nhà |
| `co_ppm` | ppm | Trạng thái | Luật an toàn CO (hầm xe) |
| `fan_speed_pct` | % | Giải thích | Tác động của thông gió lên CO2 |
| `fan_speed_pct_sp` | % | Đọc lệnh | Setpoint hiện tại, làm mốc cho gợi ý |
| `fan_state` | – | Trạng thái | Quạt chạy / dừng |
| `fan_state_cmd` | – | Đọc lệnh | Lệnh vs thực tế |
| `damper_state` | – | Giải thích | Tỷ lệ gió tươi |
| `damper_state_cmd` | – | Đọc lệnh | Lệnh vs thực tế |
| `air_flow_m3h` | m³/h | Giải thích | Lưu lượng gió thực |
| `supply_air_temp_c` | °C | Trạng thái | Ràng buộc tiện nghi |
| `supply_air_pressure_pa` | Pa | Trạng thái | Suy giảm quạt / tắc ống |
| `diff_pressure_pa` | Pa | Đích | Baseline chênh áp qua lọc → xu hướng tắc lọc |
| `filter_state` | – | Trạng thái | Cờ lọc bẩn từ BMS, dùng làm nhãn đối chiếu |
| `water_valve_state` | – | Trạng thái | Van nước lạnh cấp cho coil PAU/AHU |
| `water_valve_state_cmd` | – | Đọc lệnh | Lệnh vs thực tế |
| `cooling_tower_fan_state` | – | — | Thuộc M2 (plant), không dùng ở M4 |

**Telemetry sử dụng — §2.2 PHATTAI trạm quan trắc ngoài trời (12/12)**

| Key | Đơn vị | Vai trò | Dùng cho |
|---|---|---|---|
| `pm25`, `pm25_avg`, `pm10`, `pm1` | µg/m³ | Đích / Giải thích | Dự báo bụi ngoài trời; quyết định có nên giảm gió tươi |
| `aqi` | – | Đích | Chỉ số tổng hợp để hiển thị & ngưỡng |
| `co`, `no2`, `o3`, `so2` | µg/m³ | Giải thích | Thành phần khí ngoài trời |
| `temperature`, `humidity` | °C, % | Giải thích | Tải ẩn/hiện của gió tươi (dùng chung với M1, M2) |
| `wind_speed` | m/s | Giải thích | Khuếch tán ô nhiễm ngoài trời |

**Telemetry bổ sung — Access control · ZKTeco**

| Key | Vai trò | Dùng cho |
|---|---|---|
| `record_count` | Giải thích | Δ theo 15 phút tại cửa phục vụ khu vực → proxy người vào |
| `door_state` | Giải thích | Cửa mở kéo dài ảnh hưởng CO2 / tải lạnh |
| `online` | Trạng thái | Loại khoảng dữ liệu khi đầu đọc mất kết nối |
| `user_count` | — | **Không dùng làm lượng người hiện diện** (xem mục 8) |

**Đại lượng dẫn xuất**
- `occupancy_proxy` = Δ `record_count` theo 15 phút tại các cửa gắn với vùng (chỉ đếm lượt vào nếu phân biệt được hướng).
- `co2_rate` = dCO2/dt; mô hình cân bằng khối lượng đơn giản: `dCO2/dt ≈ k1·occupancy − k2·air_flow_m3h·(CO2_in − CO2_out)`.
- `filter_dp_trend` = độ dốc `diff_pressure_pa` tại cùng `fan_speed_pct`.
- `indoor_outdoor_ratio` = chỉ số trong nhà / ngoài trời (nếu có cảm biến bụi trong nhà).

**Output cụ thể**
- Dự báo `co2_ppm` 15–30 phút theo vùng; dự báo `pm25`, `aqi` ngoài trời 1–3h.
- Bất thường: CO2 cao khi quạt chạy (damper kẹt / lưu lượng thấp), quạt chạy khi vùng vắng người, lọc tắc dần, CO vượt ngưỡng.
- Gợi ý: tăng `fan_speed_pct` trước khi CO2 vượt ngưỡng; giảm gió tươi khi `pm25` ngoài trời cao nhưng CO2 còn an toàn; giảm quạt vùng vắng người; lịch thay lọc.

---

## 7. Ma trận key × module

✔ = key chính · ○ = bổ trợ / tùy chọn

| Nhóm | Key | M1 Điện | M2 Chiller | M3 Nước | M4 Không khí |
|---|---|:-:|:-:|:-:|:-:|
| Power meter | `power_active_kw`, `energy_active_kwh_total` | ✔ | ○ | ○ | |
| Power meter | `power_factor`, `energy_reactive_kvarh_total`, `frequency_hz` | ✔ | | | |
| Power meter | `voltage_v`, `voltage_l1_v`, `voltage_l2_v`, `voltage_l3_v` | ✔ | | | |
| Power meter | `current_a`, `current_l1_a`, `current_l2_a`, `current_l3_a` | ✔ | | | |
| Chiller | 9 key nhóm chiller | | ✔ | | |
| NORIS | `active_energy_kwh`, `active_power_kw`, `voltage_l1..l3` | ○ | ✔ | | |
| NORIS | `cooling_load_kw`, `chw_supply_temp`, `chw_return_temp`, `evaporator_pressure`, `compressor_status`, `op_mode`, `run_hours` | | ✔ | | |
| NORIS | `water_volume_m3` | | ○ | ○ | |
| Air side | `cooling_tower_fan_state` | | ✔ | | |
| Air side | `supply_air_temp_c`, `fan_state` | | ○ | | ✔ |
| Air side | 14 key còn lại | | | | ✔ |
| Nước | 20 key nhóm nước | | | ✔ | |
| Nước | `pump_state`, `pump_speed_pct`, `water_flow_lpm` | | ○ | ✔ | |
| PHATTAI | `temperature`, `humidity` | ○ | ✔ | | ✔ |
| PHATTAI | 10 key còn lại | | | | ✔ |
| ZKTeco | `record_count` | ○ | ○ | ○ | ✔ |
| ZKTeco | `door_state`, `online` | | | | ○ |
| Dùng chung | `fault`, `mode` | ○ | ✔ | ✔ | ○ |
| Dùng chung | `state_on_off` | ○ | | ○ | |
| Dùng chung | `lighting_knx_state` | ○ | | | |
| Kết nối | `reachable` (ICMP của gateway/đồng hồ) | ○ | ○ | ○ | ○ |

`reachable` chỉ dùng cho chất lượng dữ liệu: phân biệt "đồng hồ mất kết nối" với "tiêu thụ bằng 0".

### 7.1 Tổng hợp số key

| Nguồn | Tổng key trong v1.13 | Dùng trong bài toán A |
|---|---:|---:|
| Power meter | 13 | 13 |
| HVAC · chiller | 9 | 9 |
| HVAC · air side | 16 | 16 |
| Nước / bơm / van / bồn | 20 | 20 |
| §2.1 Siemens NORIS | 13 | 13 |
| §2.2 PHATTAI | 12 | 12 |
| Dùng chung | 6 | 5 (trừ `lighting_knx_state_cmd`) |
| Access control · ZKTeco | 11 | 3 (`record_count`, `door_state`, `online`) |
| ICMP ping | 3 | 1 (`reachable`) |
| Attribute | 16 | 5 |

### 7.2 Không dùng trong bài toán A

- Host / gateway compute (20), Kết nối cellular/WLAN (9), GPS (6), Camera Hik ISAPI (8), Báo cháy Notifier (9), Thang máy Schindler (2) → thuộc bài toán B hoặc không liên quan.
- Tất cả key `*_cmd`, `*_sp` **không bao giờ được ghi**; chỉ đọc như bảng ở mục 6.

---

## 8. Gap dữ liệu & câu hỏi cần chốt

| # | Vấn đề | Ảnh hưởng | Việc cần làm | Chặn |
|---|---|---|---|---|
| G1 | Điện / chiller chưa có luồng real-time trong DMP, phải qua NORIS | M1, M2 | Chốt: NORIS có đủ tần suất (≤ 15 phút) và độ trễ chấp nhận được không, hay phải lắp / tích hợp thêm đồng hồ | **Chặn cả bài toán A — làm đầu tiên** |
| G2 | Hai bộ tên key cho cùng đại lượng (DMP chuẩn vs NORIS) | M1, M2 | Lập bảng mapping + kiểm tra đơn vị, dấu, counter | Chặn bước chuẩn hóa |
| G3 | Không có công suất điện riêng từng chiller trong nhóm chiller chuẩn | M2 | Xác nhận `active_power_kw` NORIS là theo từng máy hay cả plant | Chặn mô hình hiệu suất từng máy |
| G4 | `user_count` là **số người dùng đã đăng ký** trên máy chấm công, không phải số người đang có mặt | M4 (đề tài 7) | Dùng Δ `record_count`; xác nhận `record_count` có bị xóa / quay vòng khi chạm `record_capacity` không | Chặn dự báo CO2 theo người |
| G5 | Mapping cửa ZKTeco ↔ vùng CO2 ↔ PAU chưa có | M4 | Khảo sát hiện trường, lập bảng quan hệ | Chặn M4 |
| G6 | `tank_level_*` chỉ 3 mức rời rạc | M3 | Chấp nhận ước lượng rời rạc; nếu cần chính xác đề xuất cảm biến mức liên tục (ngoài phạm vi) | Không chặn |
| G7 | Không có lưu lượng nước lạnh riêng cho vòng chiller | M2 | Xác nhận `water_flow_lpm` có đồng hồ nào nằm trên vòng CHW không | Không chặn (có `cooling_kw`) |
| G8 | Thiếu nhãn sự cố thật để đánh giá bất thường | Tất cả | Thu nhật ký bảo trì / sự cố 6–12 tháng gần nhất | Chặn đánh giá precision |
| G9 | Độ dài lịch sử dữ liệu | Tất cả | Kiểm tra mỗi key có tối thiểu bao nhiêu tháng; baseline theo mùa cần ≥ 1 năm, tạm dùng ≥ 8 tuần + điều kiện nhiệt độ | Không chặn GĐ1 |

---

## 9. Tiêu chí nghiệm thu (đề xuất, cần thống nhất với mentor)

| Hạng mục | Chỉ số | Mục tiêu đề xuất |
|---|---|---|
| Chất lượng dữ liệu | Tỷ lệ mẫu hợp lệ sau chuẩn hóa | Báo cáo được cho 100% thiết bị trong phạm vi |
| Luật an toàn (M3) | Phát hiện đúng các kịch bản chạy khô / tràn / van kẹt đã tái hiện hoặc có trong log | Bỏ sót = 0 trên tập kịch bản |
| Bất thường | Precision trên tập sự kiện có nhãn | ≥ 80% |
| Dự báo phụ tải điện 24h | MAPE (giờ vận hành) | ≤ 10–15% |
| Dự báo tải lạnh 24h | MAPE (giờ vận hành) | ≤ 15% |
| Dự báo CO2 30 phút | MAE | ≤ 100 ppm |
| Gợi ý | Tiết kiệm ước tính (kWh, VND) trên dữ liệu lịch sử (backtest), có ghi rõ giả định | Có số liệu cho M1, M2 |
| Vận hành | Mỗi cảnh báo / gợi ý có bằng chứng (key, thời gian, baseline) | 100% |

---

## 10. Lộ trình

| GĐ | Nội dung | Module | Đầu ra chính |
|---|---|---|---|
| GĐ0 (tuần 1–2) | Chốt G1–G3; kiểm kê key thực có dữ liệu, tần suất, độ dài lịch sử; lập mapping NORIS; thu calendar, biểu giá | Tất cả | Báo cáo kiểm kê dữ liệu, bảng mapping |
| GĐ1 (tuần 3–6) | Chuẩn hóa; **luật an toàn M3 lên trước**; baseline 3 lớp; phát hiện bất thường | M1, M3 trước; M2, M4 theo sau | O1, O2, dashboard baseline |
| GĐ2 (tuần 7–10) | Dự báo phụ tải điện, tải lạnh, nhu cầu nước, CO2; mô hình hiệu suất chiller | M1, M2, M3, M4 | O3, đường cong hiệu suất |
| GĐ3 (tuần 11–14) | Gợi ý tối ưu theo giá điện có ràng buộc; backtest tiết kiệm; báo cáo | M2 (chính), M3, M4 | O4, O5, báo cáo tiết kiệm ước tính |

**Phân công theo module:** M1 Điện — Đức · M2 Chiller — Đôn · M3 Nước — Bảo · M4 Không khí/CO2 — thành viên còn lại. Framework (chuẩn hóa, baseline, định dạng output) làm chung, thống nhất interface từ GĐ0.
