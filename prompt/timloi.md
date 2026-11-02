
# Giá trị linh kiện trong schematic và đánh giá

Tôi đọc trực tiếp từ schematic: xuất netlist ra để có giá trị và kết nối thực tế của từng chân, không dựa vào file BOM.

**Kết luận chung:** phần lớn giá trị đã đúng. Có **3 vấn đề nên sửa**:

- Đầu ra của 2 LDO không có tụ ngay tại chân OUT.
- Tín hiệu điều khiển WS2812 3,3V chỉ vừa đủ ngưỡng của LED khi LED chạy 5V.
- Chân strapping IO2 đang thả nổi.

Ngoài ra có vài điểm nhỏ: C16 ghi sai mã linh kiện, tổng tụ trên đường 5V vượt khuyến nghị của chuẩn USB, và tụ 0402 bị sụt điện dung khi có điện áp DC.

## 1. Đầu vào USB-C và đường VBUS / +5V

| Linh kiện        | Giá trị                    | Vị trí                                                | Đánh giá                                                                                                                                                   |
| ----------------- | ---------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R4, R5            | 5,1k                         | Từ CC1 và CC2 xuống GND, mỗi chân một điện trở | ✅ Đúng chuẩn của thiết bị nhận nguồn USB-C, nên sạc Type-C sẽ cấp 5V                                                                             |
| D1                | ESDALC5-1BM2                 | TVS chống tĩnh điện trên VBUS                      | ⚠️ Điện áp làm việc khoảng 5V, trong khi VBUS có thể lên 5,25–5,5V, nên diode bị dòng rò ở sát ngưỡng. Nên chọn TVS có VRWM ≥ 5,5–6V |
| F1                | PPTC 750 mA / 8V             | VBUS → C11                                             | ✅ Dư cho tải dưới 500 mA                                                                                                                                 |
| C11, C12          | 1µ + 100n                   | Sau cầu chì                                           | ✅                                                                                                                                                            |
| FB2               | 1k@100MHz (1206)             | Sau cầu chì → +5V                                    | ✅ Lọc nhiễu tốt. Cần kiểm tra dòng định mức ≥ 1A                                                                                                   |
| C15               | 100µ / 16V tantalum polymer | +5V                                                     | ✅ 16V cho đường 5V là đủ dư                                                                                                                           |
| C23               | 47µ / 16V điện phân      | +5V                                                     | ✅                                                                                                                                                            |
| C1, C19, C21, C22 | 4,7µ, 1µ, 100n, 100n       | +5V                                                     | ✅                                                                                                                                                            |

⚠️ **Tổng tụ trên đường 5V vào khoảng 155 µF.** Chuẩn USB 2.0 khuyến nghị **≤ 10 µF** nếu không có mạch hạn dòng khởi động. Thực tế board vẫn chạy được, nhưng lúc cắm sẽ có xung dòng lớn: có thể làm cổng USB của laptop báo quá dòng, hoặc làm cầu chì PPTC nóng lên. Cách khắc phục: bỏ C23 (47 µF), hoặc giảm C15 xuống 22–47 µF.

## 2. Bộ hạ áp TLV62569A tạo +3V3 (cấp cho ESP32)

| Linh kiện   | Giá trị                                           | Đánh giá                                                                                                                                                                                                        |
| ------------ | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| R6 + R7 / R8 | 270k + 180k / 100k                                  | ✅ Vout = 0,6 × (1 + 450k/100k) = **3,30 V**, chính xác                                                                                                                                                   |
| C10          | 10p (DNP, tụ bù song song điện trở hồi tiếp) | ✅ Để trống là hợp lý, khi cần mới hàn vào                                                                                                                                                               |
| JP1          | 0Ω, EN nối lên +5V                               | ✅ Bộ hạ áp luôn bật. Gỡ JP1 thì ngắt được                                                                                                                                                              |
| L1           | 1 µH, Isat 9,6 A (XEL4020)                         | ✅ Nằm trong dải TI cho phép. Giá trị phổ biến là 2,2 µH, nên 1 µH cho gợn dòng lớn hơn nhưng phản hồi tải nhanh hơn. Dòng 9,6 A là rất dư                                                 |
| C1           | 4,7 µF 0402 / 10V (tụ đầu vào)                 | ⚠️ Giá trị đúng khuyến nghị, nhưng tụ 0402 ở 5V chỉ còn khoảng 1,5–2 µF thực tế. Nên đổi sang 10 µF loại 0603 hoặc 0805                                                                    |
| C6, C29      | 2× 22 µF 0402 / 10V (tụ đầu ra)                | ⚠️ Ở 3,3V mỗi tụ chỉ còn khoảng 6–9 µF. Tổng vẫn đạt mức tối thiểu 10 µF nên chạy được, nhưng ESP32 phát WiFi kéo dòng đỉnh khoảng 350 mA. Đổi sang 0603 hoặc 0805 sẽ chắc hơn |
| C31, C33     | 100n + 1µ                                          | ✅ Tụ lọc tại chân 3V3 của module                                                                                                                                                                             |

## 3. Nguồn cho cảm biến: TPS7A20 (+3V3_SENS) và TLV75533 (+3V3_SCD)

| Linh kiện                   | Giá trị                                   | Đánh giá                                                                                                                                                    |
| ---------------------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R11 + C14 + C17              | 36Ω + 47µ + 1µ (lọc RC trước TPS7A20) | ✅ Tần số cắt khoảng 92 Hz, lọc nhiễu tốt. Với tải khoảng 5–8 mA thì điện áp rơi chỉ khoảng 0,2–0,3V, đầu vào LDO vẫn còn khoảng 4,6V |
| R13 + C27 + C28              | 0Ω + 47µ + 1µ (trước TLV75533)         | ✅ Dùng 0Ω là đúng, vì SCD41 kéo dòng đỉnh khoảng 200 mA, không đặt điện trở 36Ω ở đây được                                            |
| JP2, JP3                     | 0Ω, EN của 2 LDO                          | ✅                                                                                                                                                             |
| FB3, FB1                     | 1k@100MHz (0603) đặt sau đầu ra LDO     | ❗ Xem bên dưới                                                                                                                                             |
| C2, C3, C5, C7, C8, C20, C24 | 100n và 1µ trên +3V3_SENS                | ✅ Đủ tụ lọc cho từng cảm biến                                                                                                                          |
| C13, C18, C32, C34           | 100n và 1µ trên +3V3_SCD                 | ✅                                                                                                                                                             |

❗ **Đầu ra của hai LDO không có tụ ngay tại chân OUT.** Netlist cho thấy:

- `Net-(U7-OUT)` chỉ gồm U7 và FB3.
- `Net-(U9-OUT)` chỉ gồm U9 và FB1.

Toàn bộ tụ nằm **sau** hạt ferrite. Ferrite có trở kháng cao ở tần số cao, nên khi nhìn từ phía LDO, tụ đầu ra coi như bị tách ra. Mạch hồi tiếp của LDO dễ **dao động hoặc mất ổn định**, nhất là khi SCD41 kéo xung dòng khoảng 200 mA mỗi lần đo. TI yêu cầu tụ ≥ 1 µF đặt ngay tại chân OUT.

Cách sửa, chọn một trong hai:

- Thêm một tụ 1 µF (0402/0603) ngay tại chân OUT của U7 và một tụ tại chân OUT của U9, nằm trước ferrite.
- Hoặc thay FB1 và FB3 bằng điện trở 0Ω.

## 4. ESP32-C3: EN, BOOT và chân strapping

| Linh kiện      | Giá trị                             | Đánh giá                                                                                                                                                                                                                |
| --------------- | ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R14 + C25 + C26 | 10k + 100n + 1µ trên EN             | ✅ τ ≈ 11 ms, đúng khuyến nghị của Espressif (10k / 1µF)                                                                                                                                                           |
| R15 + C30       | 10k + 100n trên IO9 (BOOT)           | ✅ τ = 1 ms, ngắn hơn nhiều so với độ trễ của EN (11 ms), nên IO9 đã ổn định trước khi chip chạy                                                                                                         |
| SW1, SW2        | Nút nhấn kéo xuống GND            | ✅                                                                                                                                                                                                                         |
| IO8 (SCL)       | Kéo lên bằng R2 4,7k về +3V3_SENS | ✅ Ở mức cao lúc khởi động, đúng yêu cầu. Lưu ý: nếu gỡ JP2 (tắt +3V3_SENS), IO8 mất điện trở kéo lên và chế độ nạp code qua IO9 có thể lỗi                                                   |
| IO2             | Chỉ nối với TP11                   | ⚠️ **IO2 là chân strapping, Espressif khuyến nghị giữ mức cao lúc khởi động.** Nên thêm 10k kéo lên +3V3. Hiện tại thường vẫn chạy, nhưng nhiễu có thể gây lỗi khởi động ngẫu nhiên |
| IO0, IO1        | Chỉ nối TP9 và TP10                | ✅ Không phải chân strapping, nên để trống được                                                                                                                                                                  |

## 5. USB data

| Linh kiện | Giá trị                        | Đánh giá                                                                       |
| ---------- | -------------------------------- | --------------------------------------------------------------------------------- |
| R3, R9     | 22Ω nối tiếp trên D+ và D− | ✅ Giảm phản xạ tín hiệu, đúng thông lệ                                  |
| C4, C9     | 10p (DNP)                        | ✅ Để trống là đúng. Nếu lắp vào sẽ làm méo tín hiệu USB Full-Speed |
| U4         | TPD2EUSB30                       | ✅ Chống tĩnh điện cho đường data                                          |

## 6. LED

| Linh kiện | Giá trị                                        | Đánh giá                                                                                                                                                                                       |
| ---------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D2 + R10   | LED đỏ + 330Ω (IO5)                           | ✅ Dòng qua LED ≈ (3,3 − 2,0)/330 ≈ **4 mA**, đủ sáng                                                                                                                                |
| D3 + R12   | WS2812B-2020 chạy 5V, chân DIN qua 330Ω (IO6) | ⚠️ Khi LED chạy 5V, ngưỡng mức cao của DIN là 0,7 × 5V = **3,5V**, trong khi ESP32 chỉ xuất 3,3V. Nhiều con chạy được, nhưng có con bị sai màu hoặc chớp ngẫu nhiên |

Cách khắc phục cho WS2812, chọn một:

- Chuyển mức bằng 74AHCT1G125.
- Cấp nguồn cho LED qua một diode, để VDD còn khoảng 4,3V. Khi đó ngưỡng chỉ còn khoảng 3,0V.
- Hoặc cấp LED từ +3V3. Bản 2020 chạy được ở 3,3–3,7V, nhưng màu xanh dương sẽ hơi tối.

Nên đặt thêm 100 nF sát chân VDD của D3. C21 và C22 trên +5V có thể đảm nhận nếu chúng nằm sát LED.

## 7. Bus I2C và cảm biến

| Linh kiện     | Giá trị                                                      | Đánh giá                                                     |
| -------------- | -------------------------------------------------------------- | --------------------------------------------------------------- |
| R1, R2         | 4,7k kéo lên +3V3_SENS                                       | ✅ Chạy tốt ở 100–400 kHz với 4 cảm biến và cổng Qwiic |
| Địa chỉ I2C | SHT40 0x44, BMP280 0x76 (SDO nối GND), SGP41 0x59, SCD41 0x62 | ✅ Không trùng địa chỉ                                     |
| U3 BMP280      | CSB nối VDDIO                                                 | ✅ Chọn chế độ I2C                                          |
| U6 SGP41       | Chân 4 nối GND                                               | ✅ Đúng hướng dẫn trong datasheet                          |

⚠️ Điện trở kéo lên của I2C lấy từ +3V3_SENS, trong khi SCD41 chạy từ +3V3_SCD, cổng Qwiic J2 lấy +3V3, và ESP32 chạy +3V3. Bình thường không sao. Nhưng nếu sau này tắt riêng một đường nguồn để tiết kiệm pin, cảm biến tắt nguồn có thể bị cấp điện ngược qua chân SDA/SCL. Cần nhớ điều này khi viết firmware.

## 8. Đo điện áp bằng ADC (IO3)

| Linh kiện | Giá trị                           | Đánh giá                                                |
| ---------- | ----------------------------------- | ---------------------------------------------------------- |
| R18, R19   | 10k / 10k, từ +3V3_SENS xuống GND | ✅ Đo được +3V3_SENS / 2 ≈ 1,65V, nằm trong dải ADC |

Cầu chia này luôn tốn khoảng **165 µA**. Với nguồn USB thì không sao. Nếu sau này chạy pin, nên đổi lên 100k / 100k và thêm 100 nF song song với R19.

## 9. Lỗi ghi chép trong schematic

- ❗ **C16:** giá trị ghi 1 µF nhưng trường MPN là `CC0402KPX5R6BB104`, tức tụ **100 nF**. Cần thống nhất lại giá trị đúng.
- **U9:** MPN `" TLV75533PDBVR"` có dấu cách ở đầu. **J1:** `"USB4110-GF-A "` có dấu cách ở cuối. Nên xóa đi, nếu không có thể không tra được mã khi đặt linh kiện.

## Thứ tự nên sửa

1. ❗ Thêm 1 µF tại chân OUT của U7 và U9, hoặc đổi FB1/FB3 thành 0Ω.
2. ⚠️ Thêm 10k kéo IO2 lên +3V3.
3. ⚠️ Xử lý mức logic cho DIN của WS2812.
4. ⚠️ Giảm tổng tụ trên 5V xuống. Đổi C1 thành 10 µF 0603, và C6/C29 sang cỡ 0603 hoặc 0805.
5. Sửa lại giá trị/MPN của C16 và các MPN thừa dấu cách. Cân nhắc đổi TVS D1 sang loại có VRWM ≥ 5,5V.

Mục 1 và 2 cần thêm linh kiện vào PCB. Nếu bạn muốn sửa, tôi có thể tìm vị trí trống trên board để đặt tụ 0402 sát chân LDO và điện trở kéo lên cho IO2.
