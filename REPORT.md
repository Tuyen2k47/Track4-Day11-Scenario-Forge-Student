MSSV: 2A202602439

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): A

## 1. Phát lại log có trung thực không? (bước A)

**Số liệu trích xuất từ `evidence/replay.json`:**
- Số kịch bản kiểm tra: `n = 400`, kịch bản bị bỏ do người lái va chạm: `dropped_human_crash_logs = 0`, số kịch bản sử dụng: `n_used = 400`.
- Va chạm khi chạy lại log phát lại (`replay`): `crashes = 101` (gồm 84 `rear`, 17 `side`, 0 `front`).
- Va chạm trong thế giới phản ứng (`reactive`): `crashes = 1` (gồm 1 `side`, 0 `rear`, 0 `front`).
- Va chạm xuất hiện ở cả hai môi trường: `crash_in_both = 0`.

**Giải thích:**
Phần lớn va chạm của "phát lại" là tông từ phía sau (`rear`: 84/101 vụ), và đây hoàn toàn là **va chạm giả (replay artifact / open-loop distribution shift)**. Trong log phát lại, các xe khác chỉ tua lại đúng quỹ đạo và vận tốc đã ghi của người lái ban đầu mà không có phản ứng tương tác. Khi planner xe ta lái thận trọng hoặc phanh sớm hơn người lái thực tế, xe bám đuôi TG trong log phát lại vẫn lao tới theo tốc độ cũ mà không hề giảm tốc, dẫn đến đâm vào đuôi xe ta. Trái lại, trong thế giới phản ứng (`reactive`), xe phía sau lái theo mô hình IDM có quan sát thấy xe ta phanh nên đã chủ động phanh theo, kết quả là 0 va chạm `rear`.

**Kịch bản `replay_only` điển hình (scenario 2):**
- Trong `replay_only[0]` (`scenario = 2`, `contact = "rear"`, `min_ttc_reactive = 2.74` s): khi xe ta giảm tốc nhường đường cho tình huống phía trước, xe bám đuôi TG (`tgT = 1.19` s) trong log phát lại không phanh và đâm đuôi xe ta. Ngược lại, trong thế giới phản ứng, TG nhận biết xe ta giảm tốc nên đã kịp thời phanh giữ khoảng cách an toàn, duy trì TTC nhỏ nhất lên tới 2.74 s và không xảy ra va chạm.

## 2. Kịch bản bất khả thi (bước B)

**Các luật trong `feasible.is_feasible` và căn cứ vật lý:**
1. *Gia tốc ngang khi chuyển làn:* $a_{\text{lat}} = (\pi / \text{cutDur})^2 \times 3.5 / 2 \le \mu g$. Chuyển làn 3.5 m theo hàm cosine trong thời gian `cutDur` đạt đỉnh gia tốc ngang tại giữa hành trình. Nếu $a_{\text{lat}} > \mu g$, lực ma sát ngang không đủ giữ xe, xe sẽ bị trượt văng mất lái (spin-out).
2. *Lực phanh sau khi nhập làn:* $\text{cutBrake} \le \mu g$. Gia tốc phanh dọc tối đa bị chặn bởi giới hạn bám của lốp trên mặt đường có hệ số ma sát $\mu$.
3. *Vận tốc người đi bộ:* $\text{pedV} \le 2.5\text{ m/s}$. Tốc độ đi bộ thông thường là 1.2–1.5 m/s; vận tốc trên 2.5 m/s (~9 km/h) là chạy nước rút, không phải hành vi đi bộ băng qua đường tự nhiên trong đô thị.
4. *Vận tốc xe cắt làn:* $0 < \text{cutV} \le 22\text{ m/s}$. Xe chuyển làn không thể đứng yên hoặc đi lùi ($\le 0$), và không thể vượt quá 22 m/s (~80 km/h) trên phố đô thị.

**Phân tích từ `evidence/hunt.json`:**
- Số kịch bản bất khả thi bị loại bỏ trong từng cuộc săn:
  - Cuộc săn `bus_bypass`: 2 kịch bản (`infeasible: 2`).
  - Cuộc săn `cut_in`: 1 kịch bản (`infeasible: 1`).
  - Cuộc săn `crosswalk_block`: 11 kịch bản (`infeasible: 11`).
  - Cuộc săn `rear_ended`: 14 kịch bản (`infeasible: 14`).
- Cuộc săn sinh nhiều kịch bản bất khả thi nhất là `rear_ended` (14 kịch bản) và `crosswalk_block` (11 kịch bản).
- *Vì sao CEM bị kéo về phía đó:* CEM chọn lọc các mẫu ưu tú (elite) dựa trên điểm nguy hiểm lớn nhất (TTC nhỏ nhất, khoảng cách dừng âm). Các kịch bản vi phạm vật lý (như phanh quá $\mu g$, xe cắt làn bất khả thi, người chạy quá nhanh) tạo ra tình huống cực đoan nhất, nên CEM dễ dàng bị "đánh lừa" và co cụm phân phối đề xuất vào vùng bất khả thi này.
- *Nếu đếm chúng là va chạm:* Xác suất va chạm ước tính sẽ bị phóng đại giả tạo (thổi phồng rủi ro), và kỹ sư sẽ lãng phí thời gian ép planner tối ưu cho những tình huống vật lý không bao giờ xuất hiện ngoài đời thực.

## 3. Monte Carlo cho bạn biết gì (bước C1)

**Số liệu trích xuất từ `evidence/mc.json`:**
- Số lần thử: `n = 3000`.
- Số va chạm ghi nhận: `crashes = 4`.
- Tỉ lệ va chạm điểm: $\hat{p} = 0.001333$ ($1.33 \times 10^{-3}$).
- Khoảng tin cậy Wilson 95%: $[0.000519, 0.003424]$ ($[5.19 \times 10^{-4}, 3.42 \times 10^{-3}]$).

**Cỡ mẫu cần thiết để khoảng tin cậy hẹp lại còn $\pm 20\%$ của $p$:**
Bán kính khoảng tin cậy xấp xỉ $1.96 \sqrt{p(1-p)/n} \le 0.20 p \implies n \approx \frac{1.96^2}{0.20^2 \times p} \approx \frac{96.04}{p}$.
Với $\hat{p} = 1.333 \times 10^{-3}$:
$$n \approx \frac{96.04}{0.001333} \approx 72\,045\text{ lần chạy}$$
(Cần khoảng 72.000 lần chạy Monte Carlo để đạt độ hẹp $\pm 20\%$).

**Các kiểu thất bại chưa xuất hiện:**
Trong 3.000 lần chạy, chỉ có kiểu `bus_bypass` xuất hiện ($k = 4$). Các kiểu còn lại hoàn toàn chưa xuất hiện ($k = 0$):
- `cut_in`: $k = 0$.
- `crosswalk_block`: $k = 0$.
- `rear_ended`: $k = 0$.
- `other`: $k = 0$.

**Con số đúng cần báo:**
Tuyệt đối **không được báo bằng 0** (vì 0 vụ trong 3.000 lần không có nghĩa là xác suất bằng 0). Con số đúng cần báo là cận trên khoảng tin cậy Wilson 95% cho $k=0, n=3000$:
$$\text{hi} = \mathbf{0.001279}\quad (1.28 \times 10^{-3})$$
(Khớp quy tắc số 3: $3/n = 3/3000 = 1.0 \times 10^{-3}$).

## 4. Importance sampling, ESS và độ suy biến (bước C3)

**Bảng số liệu tổng hợp từ `evidence/estimate.json`:**

| Kiểu thất bại | Seed 0: $p$ (hits / ess / top_share) | Seed 1: $p$ (hits / ess / top_share) | Seed 2: $p$ (hits / ess / top_share) |
|---|---|---|---|
| `bus_bypass` | $8.64 \times 10^{-4}$ (61 / 143.4 / 0.606) | $8.27 \times 10^{-4}$ (75 / 128.8 / 0.165) | $8.21 \times 10^{-4}$ (71 / 120.8 / 0.309) |
| `cut_in` | 0.0 (0 / 134.2 / 0.0) [cận: 0.025] | 0.0 (0 / 133.3 / 0.0) [cận: 0.025] | 0.0 (0 / 128.2 / 0.0) [cận: 0.025] |
| `crosswalk_block` | $5.58 \times 10^{-7}$ (54 / 118.7 / 0.176) | $1.19 \times 10^{-6}$ (64 / 117.6 / 0.221) | $1.58 \times 10^{-4}$ (59 / 121.5 / 0.962) |
| `rear_ended` | 0.0 (0 / 131.5 / 0.0) [cận: 0.025] | 0.0 (0 / 109.6 / 0.0) [cận: 0.025] | 0.0 (0 / 130.1 / 0.0) [cận: 0.025] |
| **Tổng (`total`)** | **$8.65 \times 10^{-4}$** | **$8.28 \times 10^{-4}$** | **$9.80 \times 10^{-4}$** |

- *Độ đồng thuận giữa các seed:* Đối với `bus_bypass`, cả 3 seed rất đồng thuận (dao động hẹp $8.21 \times 10^{-4} - 8.64 \times 10^{-4}$), và khớp rất tốt với Monte Carlo ($1.33 \times 10^{-3}$, cùng bậc $10^{-3}$). Đối với `crosswalk_block`, seed 0 và seed 1 cho kết quả cỡ $10^{-6}$, trong khi seed 2 vọt lên $1.58 \times 10^{-4}$ do hiện tượng suy biến trọng số.
- *Hiện tượng `top_share` lớn (> 0.5):* Xuất hiện ở seed 0 (`bus_bypass`: 0.606) và đặc biệt seed 2 (`crosswalk_block`: 0.962). Khi `top_share` = 0.962, một mẫu duy nhất mang tới 96.2% giá trị ước lượng của cả tập mẫu. Nguyên nhân là đề xuất $q$ đã rút trúng một mẫu rơi vào vùng có $p(u)$ cao bất thường so với $q(u)$, khiến trọng số $w = p/q$ phát nổ và quyết định toàn bộ con số ước lượng.
- *So sánh ngân sách (`budget`):* Tổng ngân sách của IS là `budget = 16800` lần chạy. Trong khi đó, để Monte Carlo đạt cùng độ chính xác cho xác suất hiếm cỡ $8 \times 10^{-4}$, cần khoảng $n \approx 96/p \approx 120\,000$ lần chạy. IS giúp tiết kiệm hơn 7 lần tài nguyên tính toán.
- *Phần `other`:* Trích từ `mc.json: other` ($k = 0, n = 3000, \text{cận trên } 1.28 \times 10^{-3}$). Ta không săn `other` vì đây là tập hợp các va chạm ngẫu nhiên không thuộc cơ chế xác định nào, không có hàm điểm nguy hiểm định hướng cho thuật toán CEM.

## 5. Các kiểu thất bại (bước C2 + 3D)

**Mô tả các kiểu thất bại tìm thấy (`evidence/hunt.json`):**
1. **Kiểu `bus_bypass` (tìm thấy 152 vụ):**
   - *Chuỗi sự kiện:* Xe buýt đỗ ở trạm che khuất tầm nhìn làn E2; xe C2 lách qua xe buýt với vận tốc chậm (< 6 m/s) lấn sang làn xe ta; xe ta lao tới ở tốc độ cao (~12.5 m/s) và cảm biến phát hiện muộn do xe buýt che chắn, dẫn đến va chạm góc trước bên trái của xe ta với C2.
   - *Ảnh chụp 3D:* ![Va chạm bus_bypass](shots/bus_bypass.png) *(kịch bản B-01 trong trình xem chương 9)*.
2. **Kiểu `crosswalk_block` (tìm thấy 22 vụ):**
   - *Chuỗi sự kiện:* Người đi bộ P1 bước ra từ sau xe buýt đỗ để sang đường; xe ta phát hiện người đi bộ muộn nên phanh gấp ở giới hạn bám; xe ta dừng lại an toàn không đâm vào người, nhưng mũi xe đã trườn vào nằm đè lên vạch qua đường, chặn lối đi của P1 khiến người đi bộ phải đứng lại.
   - *Ảnh chụp 3D:* ![Chặn lối crosswalk_block](shots/crosswalk_block.png) *(kịch bản C-01 trong trình xem chương 9)*.

**Kiểu không tìm thấy:**
- `cut_in`: Không tìm thấy vụ va chạm độc lập nào (các vụ va chạm C2 đều quy về `bus_bypass`). Ta tin tưởng kết luận này ở mức trung bình-cao, vì khi C2 nhập làn bình thường ở tốc độ cao, khoảng cách quan sát thông thoáng cho phép IDM của xe ta giảm tốc êm ái.
- `rear_ended`: Không tìm thấy vụ nào trong thế giới phản ứng (`reactive`), vì xe bám đuôi TG có logic giữ khoảng cách và phanh kịp thời.
- *Nếu chỉ chạy một cuộc săn chung:* CEM sẽ nhanh chóng bị co cụm (mode collapse) vào kiểu va chạm phổ biến và có gradient điểm nguy hiểm dốc nhất (`bus_bypass`), và sẽ bỏ sót hoàn toàn kiểu va chạm hiếm `crosswalk_block` (bằng chứng là cuộc săn `cut_in` chỉ tìm thấy `bus_bypass`, còn cuộc săn riêng `crosswalk_block` mới lôi được 22 vụ chặn lối vạch qua đường).

## 6. Sửa planner (bước E + 3D)

**Số liệu trích xuất từ `evidence/fix.json`:**
- Bộ tham số được chọn: `{'cutinLat': 3.3}` (tăng ngưỡng nhận diện xe cắt làn từ 2.15 m lên 3.3 m).
- Trên tập **TRAIN**:
  - `fail = 2.34e-06`, `fail_hi = 4.63e-06`, `progress = 215.1` m, đánh dấu `unseen: ['bus_bypass']`.
- Trên tập **TEST**:
  - **Bản sửa (`chosen: {'cutinLat': 3.3}`):** `fail = 2.66e-04`, `fail_hi_seen = 6.49e-04`, `fail_hi (valid) = 3.94e-02`, `unseen = ['bus_bypass']`, `progress = 214.2` m, `harsh = 7.81` khung hình.
  - **Bản gốc (`baseline`):** `fail = 2.58e-03`, `fail_hi_seen = 6.13e-03`, `fail_hi (valid) = 7.41e-03`, `unseen = []`, `progress = 214.4` m, `harsh = 8.29` khung hình.
- *So sánh hiệu quả:* Bản sửa giúp giảm tỉ lệ thất bại thực tế trên TEST gần **10 lần** (từ $2.58 \times 10^{-3}$ xuống $2.66 \times 10^{-4}$), giảm số khung hình phanh gấp (từ 8.29 xuống 7.81), trong khi quãng đường di chuyển gần như không đổi (214.2 m so với 214.4 m, chỉ giảm 0.07%, dưới trần 5%).

**Vì sao `fail_hi` của bản sửa lại lớn hơn `fail` của bản gốc dù bản sửa tốt hơn thật?**
Vì bản sửa thành công xóa sổ hoàn toàn kiểu va chạm `bus_bypass` trên mẫu TEST $n=400$ (`unseen: ['bus_bypass']`). Theo nguyên lý thống kê bảo thủ, một kiểu không hề xuất hiện trong $n=400$ lần thử độc lập không được coi là xác suất bằng 0 mà phải gán cận trên $3 / (0.2 \times n) = 3/80 = 0.0375$. Cộng dồn cận phạt này khiến `fail_hi` nhảy lên $3.94 \times 10^{-2}$. Đây là cái giá của việc chứng nhận độ an toàn với cỡ mẫu hữu hạn, không phải lỗi của policy.

**Núm chỉnh "nghe an toàn" nhưng phản tác dụng:**
- Ứng viên `cautionV: 6.0` (giảm tốc độ thận trọng khi qua vạch bị che xuống 6 m/s thay vì 7.5 m/s) nghe rất an toàn, nhưng trên TRAIN làm tỷ lệ thất bại vọt lên `fail = 2.20e-03` (tăng gấp gần 10 lần so với baseline `2.36e-04`)!
- *Giải thích phản ứng thế giới:* Khi xe ta phanh giảm tốc quá sớm và bò quá chậm ở đoạn đường trước vạch qua đường, xe C2 phía sau xe buýt thấy khoảng trống phía trước xe ta rộng hơn nên quyết định lấn làn sớm hơn; đồng thời thời điểm xe ta đến vạch qua đường bị trễ lại đúng lúc người đi bộ P1 bước ra, làm thân xe ta dừng kẹt ngay giữa vạch qua đường.

**Ảnh chụp 3D có bóng mờ:**
![Bóng mờ kịch bản sửa](shots/fix_ghost.png) *(Bóng mờ là policy gốc va chạm/chặn vạch, trong khi xe thật chạy policy mới dừng an toàn)*.

## 7. Từ mô phỏng ra đường thật

**Ba giả định quan trọng nhất trong `avsim/space.py`:**

1. **Tỉ lệ người đi bộ ngó qua xe buýt trước khi qua đường (`pedLook = 0.85`):**
   - *Dữ liệu thay thế:* Tập dữ liệu hành vi người đi bộ thực tế ghi nhận từ camera góc cao đô thị (như PIE - Pedestrian Intent and Action, JAAD).
   - *Hướng sai lệch:* Ngoài đời thực, người đi bộ (đặc biệt trẻ em hoặc người sử dụng điện thoại) chỉ ngó nghiêng cẩn thận khoảng 40–60%. Giả định 85% là quá cao, khiến mô phỏng đánh giá **quá lạc quan** về an toàn (ngoài đời thực người đi bộ sẽ bất ngờ bước ra nhiều hơn).
2. **Khoảng hở xe chuyển làn chấp nhận (`cutGap`, trung vị 12.0 m):**
   - *Dữ liệu thay thế:* Quỹ đạo chuyển làn tự nhiên từ drone trên cao (HighD dataset, NGSIM).
   - *Hướng sai lệch:* Trong giao thông đô thị đông đúc, tài xế chuyển làn thường chen vào những khe hở rất hẹp (6–8 m). Đặt trung vị 12 m khiến mô phỏng đánh giá **thấp hơn thực tế** tần suất các cú tạt đầu nguy hiểm.
3. **Xác suất đường ướt ma sát thấp (`wet = 0.20`, ma sát giảm từ 0.8 xuống 0.5):**
   - *Dữ liệu thay thế:* Thống kê khí tượng thủy văn theo mùa kết hợp dữ liệu cảm biến đo ma sát mặt đường (chỉ số trượt ABS/ESP) từ đội xe thương phẩm chạy thực tế.
   - *Hướng sai lệch:* Khi mưa to hoặc có váng dầu/vạch sơn trơn trượt, ma sát thực tế có thể tụt xuống 0.35–0.4. Giả định $\mu = 0.5$ khiến quãng đường phanh trong mô phỏng ngắn hơn thực tế, làm giảm nhẹ ước lượng nguy cơ va chạm trong điều kiện thời tiết xấu.

## 8. Giới hạn

**Ba điều mà xác suất báo cáo KHÔNG nói lên:**
1. *Bản đồ tuyến đường quá đơn giản:* Thế giới mô phỏng chỉ gồm một đoạn phố thẳng 2 làn mỗi chiều với một trạm xe buýt và một vạch qua đường. Xác suất không phản ánh rủi ro tại ngã tư, nút giao vòng xuyến, góc cua khuất tầm nhìn, hay khi có chướng ngại vật tĩnh trên mặt đường.
2. *Mô hình hành vi người lái đơn giản (IDM thuần túy):* Các xe khác chỉ tuân theo IDM chuẩn và chuyển làn cosine trơn tru, không có tài xế say xỉn, không có xe vượt ẩu lấn làn ngược chiều, không có hành vi bóp còi hay chuyển làn giật cục.
3. *Cảm biến hoàn hảo không độ trễ:* Cảm biến của xe ta nhìn xuyên suốt trong phạm vi 120 m (chỉ bị che khuất hình học bởi thân xe buýt), không có độ trễ nhận diện của mô hình AI (perception latency 200–300 ms), không có nhiễu cảm biến do ánh sáng chói hay mưa bụi bẩn ống kính.

**Nếu được thêm một tuần:**
Tôi sẽ tích hợp mô hình độ trễ cảm biến (perception delay 150–300 ms) và đo kiểm trên nút giao ngã tư có đèn tín hiệu giao thông, nhằm đánh giá rủi ro khi chuyển pha đèn vàng/đỏ và hành vi xe rẽ trái cắt đầu xe đi thẳng.

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** Dựng một xe đạp di chuyển ở mép làn ngoài E2 ($x = 80$ m, $d = -1.2$ m, $v = 4.0$ m/s), bất ngờ lạng sang trái 0.8–2.2 m vào làn E1 của xe ta trong 1.8 giây khi xe ta tiếp cận trong khoảng 10–35 m. Ngoài đời, đây là tình huống cực kỳ điển hình: người đi xe đạp đi sát lề/vạch phân làn bất ngờ đánh lái tránh nắp cống gồ ghề, ổ gà hoặc vũng nước đọng.
- **Căn cứ:** Planner của xe ta có tham số `cutinLat = 2.15` m. Khi xe đạp đi ở làn E2 lệch $-1.2$ m, khoảng cách ngang ban đầu nằm ngoài hành lang nhận diện của planner, khiến planner coi làn đường phía trước hoàn toàn thông thoáng và duy trì tốc độ tối đa $v_0 = 12.5$ m/s. Khi xe đạp lạng gấp vào làn, thời gian đáp ứng của IDM không đủ để phanh êm ái.
- **Giả định:**
  - `space`: `{"h0_x": [20.0, 140.0], "h0_swerve": [0.8, 2.2], "h0_trig": [10.0, 35.0]}` (vận tốc lạng ngang đạt đỉnh $\le 3.0\text{ m/s}^2$ tuân thủ giới hạn vật lý `BIKE_LAT_MAX`).
  - `exposure = 0.03`: Khoảng 3% chuyến đi trong đô thị gặp tình huống xe đạp lạng sang làn xe cơ giới (căn cứ theo nghiên cứu tương tác phương tiện hỗn hợp NHTSA/FHWA).
- **Kết quả theo `evidence/mine.json`:**
  - Tỉ lệ kịch bản khả thi: `accepted = 83.3%` (1.249 / 1.500 kịch bản, vượt xa ngưỡng yêu cầu 50%).
  - **Với Policy gốc (`baseline`):**
    - Tổng cộng 7 vụ va chạm: gồm **6 vụ tránh được bằng phanh hết cỡ (`avoidable`)**, **0 vụ không tránh được (`unavoidable`)**, và **1 vụ không quy được (`unattributed`)**; 0 vụ ở vận tốc thấp (`low_speed = 0`).
    - $P(\text{lỗi planner} \mid \text{rắc rối}) = \mathbf{5.37 \times 10^{-3}}$, khoảng Wilson 95%: $[\mathbf{2.46 \times 10^{-3}}, \mathbf{1.17 \times 10^{-2}}]$.
    - Tần suất mỗi chuyến đi: $\mathbf{1.61 \times 10^{-4}}$ vụ/chuyến (cận trên $3.50 \times 10^{-4}$ vụ/chuyến).
  - **Với Policy sửa (`chosen: cutinLat = 3.3`):**
    - Tổng cộng **0 vụ va chạm** (0 avoidable, 0 unavoidable, 0 unattributed).
    - $P(\text{lỗi planner} \mid \text{rắc rối}) = \mathbf{0.0}$ (cận trên $3.43 \times 10^{-3}$).
    - Tần suất mỗi chuyến đi: $\mathbf{0.0}$ (cận trên $1.03 \times 10^{-4}$ vụ/chuyến).
- **Hình:**
  ![Va chạm kịch bản xe đạp](shots/hazard_bike.png) *(Kịch bản GB-01 lúc xe đạp lạng vào làn)*.
- **Bạn tự lái thì sao:** Khi tự cầm lái trong chương 10, người lái phát hiện xe đạp đi sát vạch phân làn từ xa nên chủ động nhả chân ga và hơi nghiêng lái sang trái từ trước khi xe đạp lạng. Planner phản ứng chậm hơn vì nó chỉ phản ứng khi tâm xe đạp đã vượt qua ngưỡng `cutinLat`.
- **Cách sửa:**
  - Núm chỉnh giải quyết triệt để rắc rối này chính là **tăng `cutinLat` lên 3.0 – 3.3 m** (chính là policy được chọn ở bước E!). Việc mở rộng hành lang ngang giúp xe ta phát hiện xe đạp bắt đầu lấn làn sớm hơn gần 1 giây, đủ để IDM phanh giảm tốc an toàn mà không cần can thiệp phanh gấp.
  - Nếu không thay đổi núm chỉnh, planner cần thêm **logic dự đoán quỹ đạo (trajectory prediction)**: phát hiện vận tốc ngang ($v_{\text{lat}} > 0.3$ m/s) hướng về tim đường của các phương tiện hai bánh ở làn bên để kích hoạt chế độ nhường đường dự phòng trước khi chúng cắt qua tim làn.
