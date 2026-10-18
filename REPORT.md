# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Đỗ Mạnh Nghĩa |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602971 |
| GitHub | domanhnghiaforwork-cloud |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 2697 / 3872 (danh sách model seeds: 3872, 3873, 3874) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.253 | 20.0% | BLOCK (trượt R2, R3, R4, R6) |
| IDM | 1.544 | 0.1% | (baseline) |
| EgoMLP | 0.771 .. 0.778 | 20.3% .. 36.1% | BLOCK (trượt R2, R3, R4, R5, R6) |
| PercMLP | 0.627 .. 0.632 | 1.1% .. 1.5% | BLOCK (trượt R2, R3, R4, R6) |
| PercMLP+ (sau khi sửa) | 0.806 .. 0.844 | 0.2% .. 0.3% | RELEASE (vượt qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Trong Bước 0 kiểm toán dữ liệu (`data_audit.json`), ta phát hiện ba vấn đề trọng yếu:
1. Có tới 383 quỹ đạo bị trùng lặp hoàn toàn trên tổng số 9,993 quỹ đạo thô ban đầu (giữ lại 9,610 quỹ đạo sạch). Các bản ghi trùng này lại nằm ở các `Trajectory_ID` cách xa nhau, nên nếu chia train/val ngẫu nhiên thông thường sẽ gây ra hiện tượng rò rỉ dữ liệu (data leakage) nghiêm trọng.
2. Cột gia tốc xe dẫn `Acc_LV` thô hoàn toàn không khớp với thực tế: hệ số tương quan tuyến tính giữa `Acc_LV` và đạo hàm vận tốc xe dẫn $d(\text{Speed\_LV})/dt$ chỉ đạt 0.0454 (gần như bằng 0, hoàn toàn sai lệch), trái ngược với xe sau người lái `Acc_FAV` đạt độ tương quan 0.9291. Do đó, ta không thể tin cậy cột `Acc_LV` mà phải thay thế bằng cách làm mượt chuỗi vận tốc xe dẫn qua bộ lọc (`v_lead_s`) rồi mới lấy sai phân chia cho bước thời gian $dt = 0.1$ s để suy ra gia tốc chuẩn xác `a_lead`.
3. Ta bắt buộc phải chia tập dữ liệu theo quỹ đạo (trajectory) với tỷ lệ 70/30 (6,727 train / 2,883 test) thay vì chia theo từng khung hình (frame), bởi vì các khung hình trong cùng một quỹ đạo có tính tương quan chuỗi thời gian (temporal correlation) cực kỳ cao. Nếu chia theo frame, các frame lân cận của cùng một xe sẽ xuất hiện ở cả hai tập, khiến mô hình học vẹt (overfitting) nhưng vẫn đạt điểm số cao giả tạo.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Từ kết quả bước 1 (`replay.json`):
1. Bản tái cấu trúc mã nguồn (refactor) dùng float32 và thay đổi thứ tự tính toán khác biệt từng bit ở toàn bộ 600/600 đoạn (100%), nhưng lại vượt dung sai ở đúng 0/600 đoạn (`differ_beyond_tol = 0`), với độ lệch cực đại `max_abs_diff` chỉ $2.2 \times 10^{-5}$ m/s² (nhỏ hơn nhiều so với ngưỡng `atol=1e-3, rtol=1e-3`). Điều này bắt nguồn từ bản chất toán học của số thực dấu phẩy động: phép cộng float không có tính kết hợp (minh chứng bằng ví dụ `(0.1 + 0.2) + 0.3 = 0.6000000000000001 != 0.1 + (0.2 + 0.3) = 0.6`). Do đó, kiểm tra bằng bit-identical là quá máy móc và sai lầm; ta phải dùng so sánh có dung sai.
2. Việc bổ sung luật AEB khẩn cấp khi $TTC < 1.5$ s làm lệch 8/600 đoạn (cụ thể là các đoạn 55, 73, 234, 239, 260). Khi mở các đoạn này (như đoạn 55 hoặc 73), sự sai khác chỉ xuất hiện ở các khung hình mà khoảng cách gap bị thu hẹp nhanh và thời gian tới va chạm $TTC = gap / closing < 1.5$ s. Ở các khung nguy hiểm đó, AEB can thiệp ghi đè lệnh gia tốc của IDM xuống mức phanh khẩn cấp $-6.0$ m/s²; còn ở các khung hình có TTC an toàn, AEB giữ nguyên lệnh điều khiển của IDM nên kết quả hoàn toàn trùng khớp với bản golden.
3. Chế độ phát theo thời gian thực (real-time) với độ trễ xử lý ngẫu nhiên làm lộ ra 582 khung bị trễ (late) và 576 khung bị rớt (dropped) trên tổng số 59,721 khung, khiến 372/600 đoạn bị biến đổi lệnh điều khiển do phải giữ lệnh trước đó (held command). Ngược lại, chế độ phát nhanh nhất ("fastest") chỉ đơn thuần tống dữ liệu đi càng nhanh càng tốt mà không màng tới hạn thời gian (0 trễ, 0 rớt), che giấu hoàn toàn các điểm nghẽn tính toán và lỗi rớt xung của hệ thống nhúng trong môi trường vận hành thực tế.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Từ phân tích lưới kịch bản cắt vào (`scenarios.json`):
1. Lưới kịch bản được xây dựng với vận tốc cruise $v_{ego} = 10.0$ m/s, dải khoảng cách từ 2.15 m đến 72.83 m (60 mốc) và tốc độ tiếp cận $dv$ từ 0 đến 10.0 m/s (45 mốc), tạo thành lưới 2,700 ô kịch bản.
2. Ở độ trễ phản ứng $t_{react} = 0.0$ s, có 1.0% số ô trên lưới là vật lý không thể tránh va chạm (unavoidable), dù xe đạp phanh tối đa $b_{max} = 9.0$ m/s² ngay từ thời điểm đầu tiên. Khi độ trễ phản ứng của hệ thống tăng lên $t_{react} = 0.6$ s, diện tích vùng đâm bất khả thi tăng vọt lên 5.1% (tăng hơn 5 lần, tăng thêm 4.1 điểm % diện tích). Tỷ lệ va chạm của IDM khớp hoàn hảo 100% với đường biên bất khả thi này (0 ô lệch ra ngoài biên ở cả hai độ trễ), tăng tương ứng từ 1.0% lên 5.1%. Ranh giới vật lý $gap < dv \cdot t_{react} + \frac{dv^2}{2 b_{max}}$ là công cụ mấu chốt để phân định giữa một ca va chạm bất khả kháng với một lỗi do thuật toán điều khiển kém.
3. Log lái xe thực tế hoàn toàn không bao phủ được các tình huống khó: người lái trong log phanh mạnh nhất ở phân vị 99.9% chỉ đạt $3.6$ m/s², trong khi phần cứng phanh có thể đạt tới $9.0$ m/s². Có tới 5% số ô trên lưới kịch bản đòi hỏi mức giảm tốc vượt quá mức phanh gắt nhất mà con người từng đạp trong log. Hơn nữa, 614 ca phanh khai thác được từ log (chỉ chiếm 6.4% tổng số 9,610 quỹ đạo) có mức phanh trung vị p50 là $-2.5$ m/s² và p99 là $-4.2$ m/s², hoàn toàn thiếu vắng các pha xe cắt vào đột ngột ở cự ly gần hoặc phanh sát giới hạn vật lý.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh giữa đánh giá vòng hở (L2 position error) và vòng kín (Closed-loop collision rate) trên 2,883 quỹ đạo kiểm thử:
| Bộ điều khiển | L2 vòng hở (m) | Hạng L2 | Tỷ lệ va chạm vòng kín (%) | Hạng vòng kín |
|---|---|---|---|---|
| constant-velocity | 1.253 | 3 | 20.0% | 3 |
| IDM | 1.544 | 4 | 0.1% | 1 |
| EgoMLP | 0.774 | 2 | 28.0% | 4 |
| PercMLP | 0.629 | 1 | 1.3% | 2 |

Có tới 4 cặp bộ điều khiển bị đảo lộn hoàn toàn thứ tự xếp hạng giữa hai cách chấm: `('const-vel', 'IDM')`, `('const-vel', 'EgoMLP')`, `('IDM', 'EgoMLP')`, và `('IDM', 'PercMLP')`.
- IDM có điểm L2 vòng hở kém nhất (1.544 m, đứng chót bảng hạng 4) vì nó là một bộ điều khiển phản hồi kín, mục tiêu của nó là duy trì ổn định khoảng cách an toàn lâu dài theo quy luật vật lý chứ không bắt chước các biến thiên vận tốc ngẫu nhiên của từng tài xế trong dữ liệu quá khứ. Tuy nhiên, khi đưa vào tự lái vòng kín, nhờ cơ chế phản hồi cảm biến liên tục qua từng bước thời gian, IDM điều chỉnh ga/phanh cực kỳ chính xác và chỉ va chạm 0.1% (3/2883 xe, an toàn nhất hệ thống).
- Trái lại, EgoMLP đạt L2 vòng hở rất đẹp (0.774 m, tốt hơn nhiều so với constant-velocity 1.253 m) vì mạng học sâu bắt chước rất khéo quán tính vận tốc của chính nó trên tập huấn luyện. Nhưng khi chạy vòng kín, các sai số dự báo nhỏ tích lũy dần qua từng chu kỳ điều khiển (compounding errors), đẩy xe vào các trạng thái phân phối lạ (covariate shift). Nguy hiểm hơn, EgoMLP hoàn toàn mù thông tin xe trước (không có đầu vào gap và $v_{lead}$), do đó khi xe dẫn phanh gấp, EgoMLP vẫn vô tư duy trì tốc độ và gây va chạm tới 28.0% (thậm chí đâm nhiều hơn cả constant-velocity 20.0%).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Từ số liệu vòng kín qua 3 seed huấn luyện (`closedloop.json`):
1. Tỷ lệ va chạm của EgoMLP dao động cực kỳ mạnh mẽ giữa các seed: từ 20.3% (seed 3874) lên 27.7% (seed 3873) và chạm đỉnh 36.1% (seed 3872), tạo ra dải biến thiên rộng tới 15.8 điểm %. Khoảng tin cậy Wilson 95% ở seed xấu nhất là [34.3%, 37.8%], hoàn toàn tách rời và không hề có phần giao cắt với khoảng tin cậy của seed tốt nhất [18.9%, 21.8%]. Tương tự, PercMLP cũng có tỷ lệ va chạm dao động từ 1.1% đến 1.5% qua 3 seed (CI95 của seed xấu nhất là [1.1%, 2.0%]).
2. Nếu một kỹ sư kiểm thử chỉ chạy đúng một seed ngẫu nhiên, họ có thể rơi vào seed may mắn (20.3%) và vội vã đưa ra báo cáo rằng mô hình học sâu này chỉ va chạm tương đương bộ điều khiển giữ tốc độ cơ bản (20.0%), trong khi thực tế khi đổi hạt giống ngẫu nhiên, tỷ lệ va chạm có thể nhảy vọt lên 36.1% (hơn 1/3 số xe bị đâm).
3. Việc chỉ báo cáo một con số duy nhất là cực kỳ nguy hiểm trong công nghệ xe tự hành: nó che giấu tính bất định và độ nhạy cảm cao của mạng nơ-ron trước việc khởi tạo trọng số ngẫu nhiên cũng như thứ tự xáo trộn batch dữ liệu. Báo cáo dải qua các seed và khoảng tin cậy Wilson là yêu cầu bắt buộc để nhận diện các rủi ro đuôi (tail risks) trước khi cấp phép phát hành một hệ thống an toàn.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng kết quả va chạm trên lưới kịch bản cắt vào ở độ trễ phản ứng $t_{react} = 0.6$ s (`agents.json`):
| Bộ điều khiển | Xe sau phát lại (replay) | Xe sau phản ứng (reactive) | Va chạm sau: replay / reactive |
|---|---|---|---|
| const-vel | 73.4% | 73.4% | 0.0% / 0.0% |
| IDM | 70.1% | 5.1% | 64.9% / 0.0% |
| EgoMLP | 74.1% | 74.1% | 0.0% / 0.0% |
| PercMLP | 70.9% | 24.6% | 47.9% / 0.0% |

- Giá trị thời khoảng cách được hiệu chỉnh tối ưu cho tác nhân xe sau IDM là $T = 1.4$ s, đem lại khoảng cách phân phối Wasserstein W1 chỉ 0.36 so với phân phối khoảng cách thời gian của người lái thật trong log (trung vị headway thực tế là 1.85 s), chân thực hơn rất nhiều so với một tác nhân mù không bao giờ phản ứng (W1 = 0.87).
- "Va chạm giả" (fake rear collisions) xảy ra khi xe sau chỉ phát lại một cách máy móc dữ liệu hành trình trong quá khứ mà hoàn toàn không biết xe thử nghiệm (ego) phía trước đang phanh. Khi xe thử nghiệm (IDM) phát hiện tình huống xe cắt vào nguy hiểm và kích hoạt phanh giảm tốc để tránh đâm phía trước, xe sau phát lại tĩnh vẫn giữ nguyên tốc độ và tông thẳng vào đuôi của IDM. Điều này khiến IDM bị tính oan tới 64.9% va chạm từ phía sau (đẩy tổng tỷ lệ va chạm lên 70.1%).
- Khi xe sau được mô hình hóa thành tác nhân phản ứng biết quan sát và phanh theo, va chạm phía sau của IDM lập tức triệt tiêu về đúng 0.0%, và tổng va chạm rơi xuống đúng 5.1% (bằng đúng ranh giới vật lý bất khả thi). Ngược lại, constant-velocity không hề phanh nên xe sau phát lại không tông vào nó (0.0% va chạm sau). Nếu môi trường kiểm thử không có tác nhân phản ứng, hệ thống sẽ đưa ra kết luận hoàn toàn sai lệch rằng constant-velocity an toàn tương đương IDM (73.4% vs 70.1%), trừng phạt bất công những thuật toán biết phanh an toàn.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP ban đầu bị chặn (BLOCK) bởi 4 quy tắc an toàn cốt lõi:
1. R2 (`R2_closed_loop_logs`): tỷ lệ va chạm 1.5% với cận trên CI95 là 2.0% vượt quá ngưỡng cho phép (baseline 0.1% + 1.0% = 1.1%).
2. R3 (`R3_braking_slice`): tỷ lệ va chạm trên lát cắt xe dẫn phanh là 4.7% với cận trên CI95 là 9.0% (vượt ngưỡng 0.6% + 5.0% = 5.6%).
3. R4 (`R4_scenario_grid`): diện tích va chạm có thể tránh được trên lưới kịch bản lên tới 19.4% (vượt xa baseline 0.0% + 2.0% = 2.0%).
4. R6 (`R6_reactive_world`): diện tích va chạm trong thế giới có xe sau phản ứng là 24.6% (vượt baseline 5.1% + 2.0% = 7.1%).

Nguyên nhân gốc rễ gồm hai yếu tố: (a) Dữ liệu log không hề có các tình huống xe cắt vào gấp ở tốc độ tiếp cận lớn (gây rớt R4, R6); (b) Sai số bắt chước tích lũy trong vòng kín khiến xe trôi dạt khỏi phân phối huấn luyện (gây rớt R2, R3).
Ta sửa mô hình bằng hai giải pháp kết hợp:
- Bổ sung 197,220 cửa sổ dữ liệu từ 20,000 kịch bản cắt vào mô phỏng (`cutin_windows`) được gán nhãn bằng luật chuyên gia IDM, giúp mô hình học cách ứng phó trong các tình huống hiểm mà log không có.
- Áp dụng thuật toán DAgger (`dagger_windows`, 2 vòng lặp) cho mô hình tự lái trên tập log, thu thập các trạng thái xe tự đi lệch và gán nhãn bằng dự báo 3 giây của IDM, giúp mô hình học cách tự phục hồi sai số.
Sau khi sửa, `PercMLP+` đã vượt qua toàn bộ 7 quy tắc của cổng (RELEASE), giảm va chạm trên log xuống 0.2%..0.3% (cận trên CI95 0.6% <= 1.1%) và diện tích va chạm trên lưới cắt vào xuống chỉ còn 0.1% (<= 2.0%).

Tuy nhiên, PercMLP+ **tuyệt đối không tốt hơn** IDM! Nó chỉ đạt tiêu chuẩn "không kém hơn" (non-inferior) baseline theo ngưỡng cổng đặt ra. Vì PercMLP+ học hoàn toàn từ nhãn của IDM, một học trò học bắt chước không thể vượt qua năng lực của người thầy tạo ra nhãn cho nó (trên lưới tốc độ cao 20 m/s ngoài phạm vi học, IDM va chạm 0.0% trong khi PercMLP+ vẫn có thể va chạm từ 0.4% đến 6.6%).
Do đó, **chưa nên phát hành thay thế IDM ngay lập tức**. Đề xuất trước khi phát hành: (1) Triển khai thử nghiệm ở chế độ bóng (shadow mode) trên xe thật; (2) Mở rộng kiểm thử kịch bản với cảm biến có nhiễu nặng và trễ; (3) Thiết kế kiến trúc an toàn 2 tầng với IDM hoặc bộ phanh AEB cứng đóng vai trò lớp bảo vệ giám sát tối thượng (safety fallback) can thiệp ghi đè khi mạng nơ-ron đưa ra lệnh nguy hiểm.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Để có cái nhìn khách quan về việc ứng dụng thực tế, bài lab này có ít nhất ba giới hạn cốt tử làm suy giảm độ tin cậy của kết luận phát hành:
1. **Mô hình động học thuần 1 chiều dọc (1D longitudinal dynamics only):** Bài lab chỉ mô phỏng vị trí và vận tốc dọc trên một đường thẳng, hoàn toàn bỏ qua động lực học ngang (đánh lái, đổi làn, lật xe, trượt bánh khi phanh gấp trên đường cong). Trong thế giới thực, xử lý va chạm không chỉ có đạp phanh mà thường là sự kết hợp giữa phanh và đánh lái né tránh chướng ngại vật; phanh gấp trên mặt đường có độ bám kém (mưa, băng tuyết) có thể gây mất lái hoàn toàn.
2. **Giả định cảm biến hoàn hảo (No perception latency and sensor degradation):** Xe thử nghiệm nhận được giá trị khoảng cách gap và vận tốc xe dẫn một cách hoàn hảo trực tiếp từ bộ mô phỏng (hoặc chỉ thêm nhiễu Gauss đơn giản). Trong thực tế, hệ thống nhận thức (Camera, LiDAR, Radar) thường xuyên bị trễ khung hình, mất bám mục tiêu (drop tracking), che khuất tầm nhìn (occlusion), hoặc sinh ra các vật thể ảo (false positives), khiến mạng học sâu dễ đưa ra quyết định giật cục hoặc phanh ma nguy hiểm.
3. **Môi trường xe dẫn không phản ứng và thế giới mô phỏng giả định:** Xe dẫn trong các kịch bản log hoàn toàn di chuyển theo dữ liệu ghi sẵn mà không hề có bất kỳ phản ứng tương tác nào với hành vi của xe thử nghiệm. Mô hình mô phỏng giả định thế giới tuân theo các bước tính toán rời rạc lý tưởng $dt = 0.1$ s, bỏ qua độ trễ cơ cấu chấp hành thủy lực của phanh thật (thường trễ 150–300 ms) và quán tính động cơ.

**Các bước cần làm tiếp theo để thu hẹp khoảng cách với thực tế:**
- Nâng cấp môi trường thử nghiệm lên các bộ mô phỏng 3D chuyên dụng độ trung thực cao kết hợp mô phỏng cảm biến vật lý chân thực (CARLA, hardware-in-the-loop).
- Chạy thử nghiệm ở chế độ bóng (Shadow Mode) trên hàng triệu km dữ liệu vận hành thực tế của đội xe thương mại để kiểm tra tính ổn định trước khi cấp quyền can thiệp vào chân ga/phanh.
- Áp dụng kiến trúc an toàn có lớp bảo vệ hình thức (Formal Safety Shielding / Control Barrier Functions) sử dụng các quy tắc vật lý bất biến để đảm bảo xe luôn nằm trong bao lơn an toàn bất kể đầu ra của mô hình AI.
