# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Vu Duc Thien |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602437 |
| GitHub | vuthien3002-sys |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split seed 1527 / model seed 8044 (5 seed: 8044–8048) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 (mặc định) |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.273 | 19.39 (CI95 [18.0, 20.9]) | BLOCK (R2, R3, R4, R6) |
| IDM | 1.525 | 0.07 (2/2883, CI95 [0.02, 0.25]) | (baseline) |
| EgoMLP | 0.767 .. 0.774 | 15.99 .. 26.71 | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0.627 .. 0.631 | 0.49 .. 1.32 | BLOCK (R2, R3, R4, R6) |
| PercMLP+ (sau khi sửa) | 0.808 ± 0.013 | 0.17 .. 0.24 | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Bước 0 cho thấy ba vấn đề. (1) Cột `Acc_LV` gần như không liên quan đến đạo hàm của `Speed_LV`: tương quan chỉ **0.045**, trong khi cột tương ứng của xe theo sau là **0.929**. Vì vậy `Acc_LV` không dùng được; mình dùng gia tốc xe dẫn suy ra từ `Speed_LV` **đã làm mượt** (`logs.a_lead`). Với gia tốc này, phân vị 0,1 % của xe dẫn chỉ là **−2.77 m/s²**. (2) Có **383** quỹ đạo trùng hệt nhau bị loại (9993 → **9610** quỹ đạo). Các bản trùng nằm ở những id cách xa nhau, nên chia ngẫu nhiên sẽ đưa cùng một quỹ đạo vào cả train và test, làm điểm test đẹp giả. (3) Phải chia theo **quỹ đạo**: các khung liền nhau (10 Hz, mỗi quỹ đạo 70–109 bước) gần như giống hệt nhau. Nếu chia theo khung hình, mô hình sẽ "thấy" gần như cùng một tình huống khi test, nên điểm test không đo được khả năng tổng quát.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã cho **0/600** đoạn khác nhau. Bản refactor float32 khác bit ở cả **600/600** đoạn nhưng vượt dung sai ở **0** đoạn: sai lệch lớn nhất chỉ **4.0e-4** m/s², dưới atol = rtol = 1e-3. Phép cộng số thực dấu phẩy động không có tính kết hợp ((0.1+0.2)+0.3 = 0.6000000000000001 ≠ 0.6), nên so khớp từng bit là phép thử sai. Dung sai cần lớn hơn nhiễu làm tròn nhưng nhỏ hơn một thay đổi hành vi thật. Thêm AEB làm lệch **8/600** đoạn (62, 146, 247, 319, 355, …). Mở đoạn 62 bằng `peek.py`: chỉ các khung 98–101 lệch, đúng những khung có TTC 1.43–1.49 s, dưới ngưỡng AEB 1.5 s. Ở đó lệnh IDM cũ (−5.4 .. −5.9 m/s²) bị kẹp xuống −6.0; mọi khung khác có TTC ≥ 1.5 s nên giữ nguyên. Phát nhanh nhất có **0** khung trễ và **0** khung rớt vì không có hạn chót. Phát theo thời gian thật lộ ra **587** khung trễ + **579** khung rớt trên **60196** khung, làm đổi **373/600** đoạn: một lần xử lý chậm 140 ms vượt hạn 100 ms thì lệnh cũ bị giữ, mà chế độ nhanh nhất không bao giờ thấy được điều đó.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới cắt vào: tốc độ cruise **10 m/s**, khoảng cách **2.76 .. 59.3 m** (p2..p98 của log), tốc độ tiến gần 0 .. 10 m/s, **60 × 45 = 2700** ô. Biên vật lý (phanh 9 m/s² sau thời gian trễ) cho **1.0 %** ô không ai tránh được khi trễ 0 s, tăng lên **5.1 %** khi trễ 0.6 s. Bộ mô phỏng khớp với hàm `unavoidable` của mình ở cả hai độ trễ (**0** ô lệch xa biên). Vùng đâm của IDM tăng từ 1.0 % → 5.1 %, và **trùng đúng** vùng bất khả thi: phần đâm "do bộ điều khiển" (`attributable_area`) của IDM là **0.0 %**. Biên này giúp phân biệt "ai cũng đâm" với "bộ điều khiển đâm". Log không phủ vùng khó: người lái trong log phanh mạnh nhất cỡ **3.6 m/s²** (p99.9), trong khi bộ chấp hành làm được 9. **5.2 %** lưới đòi phanh mạnh hơn mức mọi người lái trong log từng làm. Có **614** ca xe dẫn phanh được khai thác (6.4 % quỹ đạo), nhưng đỉnh phanh của chúng chỉ ở mức p50 **−2.5**, p99 **−4.2 m/s²**, và log hoàn toàn không có ca cắt vào. Mô hình học từ log chưa từng thấy góc trên của lưới.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?


| Bộ điều khiển | L2 vòng hở (m) | Hạng L2 | Va chạm vòng kín (TB qua seed) | Hạng va chạm |
|---|---|---|---|---|
| PercMLP | 0.629 | 1 | 0.95 % | 2 |
| EgoMLP | 0.770 | 2 | 19.2 % | 3 |
| const-vel | 1.273 | 3 | 19.4 % | 4 |
| IDM | 1.525 | 4 | 0.07 % | 1 |

Có ba cặp đổi thứ tự: (const-vel, IDM), (IDM, EgoMLP), (IDM, PercMLP). IDM xếp **cuối** về L2 nhưng **đầu** về va chạm. L2 chỉ đo mức giống quỹ đạo người lái trong log; IDM là một luật riêng với T, s0, b của nó, nên dự báo của nó lệch người lái 1.525 m. Nhưng trong vòng kín, IDM luôn phản ứng theo khoảng cách và tốc độ xe dẫn **ở trạng thái nó thật sự đang ở**, nên tự sửa sai và gần như không đâm (2/2883). EgoMLP có L2 tốt hơn constant-velocity (0.770 so với 1.273) vì nó học được cách người lái thường tăng giảm tốc. Nhưng nó chỉ nhìn tốc độ của chính nó, **không thấy khoảng cách và tốc độ xe dẫn**, nên không thể biết xe dẫn đang phanh. Trên lát cắt xe dẫn phanh, L2 của nó tăng lên 1.119 m và tỷ lệ đâm là 50.0 % (seed xấu nhất). Trung bình trên mọi cửa sổ che mất điều này, vì lát cắt chỉ chiếm 4.9 % cửa sổ test.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP dao động từ **15.99 %** (seed 8046, Wilson CI95 [14.7, 17.4]) đến **26.71 %** (seed 8044, CI95 [25.1, 28.4]) trên cùng 2883 quỹ đạo test, chênh gần 11 điểm. Hai khoảng tin cậy **không chồng nhau**. Khoảng Wilson chỉ đo nhiễu do **mẫu test hữu hạn** (rộng khoảng 3 điểm với n = 2883), không đo được độ dao động do **seed huấn luyện**, và dao động seed ở đây lớn hơn nhiều. Nếu chỉ chạy một seed, mình có thể báo "EgoMLP đâm 16 %" hay "27 %" tuỳ may rủi, và cả hai đều có CI hẹp trông rất chắc chắn. Với PercMLP còn nguy hiểm hơn: dải là **0.49 .. 1.32 %**. Seed 8045 một mình có cận trên **0.8 %**, ≤ 0.07 % + 1 điểm, tức là **qua** R2. Seed xấu nhất 8048 có cận trên **1.8 %** và **rớt**. Một lần chạy may mắn có thể đưa một mô hình chưa đạt qua cổng. Vì vậy cổng chấm theo seed xấu nhất và có R5 (dải ≤ 1 điểm).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Người lái trong log giữ khoảng thời gian trung vị **1.87 s**. Hiệu chỉnh IDM cho xe sau chọn được **T = 1.4 s**, với khoảng cách Wasserstein đến phân bố khoảng thời gian của người là **0.35**. Một xe không bao giờ phản ứng cho **0.87**, nên xe sau IDM chân thực hơn rõ rệt. Lưới cắt vào, trễ 0.6 s, tỷ lệ ô có va chạm bất kỳ:

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm phía sau (phát lại / phản ứng) |
|---|---|---|---|
| const-vel | 73.4 % | 73.4 % | 0.0 % / 0.0 % |
| IDM | 70.1 % | 5.1 % | 64.9 % / 0.0 % |
| EgoMLP | 70.3 % | 70.3 % | 0.0 % / 0.0 % |
| PercMLP | 77.4 % | 18.8 % | 59.9 % / 0.0 % |
| PercMLP+ | 69.4 % | 5.6 % | 63.8 % / 0.0 % |

Khi xe sau chỉ phát lại, nó chạy theo log và không biết xe thử vừa phanh vì xe cắt vào, nên đâm vào đuôi: **64.9 %** lưới là "va chạm giả" của IDM. Đó là lỗi của thế giới phát lại, không phải của IDM. Hệ quả là kết luận bị đảo: với xe sau phát lại, IDM (70.1 %) trông tệ ngang constant-velocity (73.4 %) và EgoMLP (70.3 %). Hai bộ này "không bị đâm từ sau" chỉ vì **chúng không phanh**, tức là phép đo thưởng cho hành vi nguy hiểm. Với xe sau phản ứng, va chạm phía sau về **0.0 %**, và IDM chỉ còn **5.1 %**, đúng bằng vùng bất khả thi.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi bốn quy tắc: **R2** (seed xấu nhất 1.3 %, cận trên 1.8 % > 0.07 % + 1 điểm), **R3** (lát cắt phanh 6.0 %, n = 182, cận trên 10.5 % > 0 % + 5 điểm), **R4** (13.6 % lưới cắt vào đâm dù tránh được, so với 0.0 % + 2 điểm), **R6** (18.8 % so với 5.1 % + 2 điểm khi xe sau phản ứng). Có hai nguyên nhân gốc. (a) **Kịch bản log không chứa** (R4, R6): log không có ca cắt vào và không có phanh mạnh hơn 3.6 m/s². Mình sửa bằng **20000** ca cắt vào mô phỏng (**197020** cửa sổ), gắn nhãn bằng việc IDM làm ở đó. (b) **Sai số tích luỹ** (R2, R3): mô hình chỉ học ở các trạng thái người lái đi qua; khi tự lái, sai số nhỏ đưa nó vào trạng thái lạ và lỗi chồng lên nhau. **DAgger** (2 vòng, khoảng 895k cửa sổ) gắn nhãn IDM ngay tại các trạng thái mô hình tự đi vào. Sau khi sửa, PercMLP+ qua cả 7 quy tắc: R2 0.2 % (cận trên 0.5 %), R3 0.0 % (cận trên 2.1 %), R4 0.4 %, R5 0.2 .. 0.2 %, R6 5.6 %. Nhưng nó **không tốt hơn** IDM, chỉ **không kém hơn trong biên** cho phép: trên log 0.17–0.24 % so với 0.07 %, lưới 0.4 % so với 0.0 %, thế giới phản ứng 5.6 % so với 5.1 %, và ở tốc độ chưa thấy 20 m/s là 0.4–1.5 % so với 0.0 %. Nhãn của nó **đến từ IDM**, nên về nguyên tắc nó không thể vượt thầy. L2 vòng hở của nó còn **tệ hơn** PercMLP (0.629 → 0.808 m) dù vòng kín tốt hơn hẳn, nên điểm vòng hở không dùng để chọn mô hình được. Mình **không** đề xuất thay IDM ngay. Trước tiên cần chạy PercMLP+ ở chế độ bóng (shadow mode) cạnh IDM, mở rộng lưới (tốc độ cao hơn, nhiễu cảm biến, độ trễ), và nêu rõ lợi ích nào (êm hơn? giống người hơn?) đủ để đánh đổi một rủi ro va chạm cao hơn IDM.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(1) **Thế giới mô phỏng là giả định của chính chúng ta, và vòng lặp tự tham chiếu**: nhãn của PercMLP+, xe sau phản ứng và biên "bất khả thi" đều dựa trên IDM hoặc vật lý 1 chiều do ta chọn (phanh tối đa 9 m/s², xe dẫn giữ tốc độ). Việc PercMLP+ "qua cổng" chỉ nói rằng nó giống IDM trong thế giới của IDM; sai giả định thì kết luận sai theo. (2) **Một chiều dọc, không cảm biến**: không có chuyển làn, không có lái ngang, nhận thức gần như hoàn hảo (thử nhiễu chỉ ±1 m khoảng cách, ±0.5 m/s tốc độ). Các lỗi phát hiện chậm, mất bám mục tiêu hay độ trễ hệ thống thật, vốn là nguồn tai nạn lớn, đều chưa được thử. (3) **Xe dẫn trong log không phản ứng, và dữ liệu hẹp**: chỉ có Argoverse 2 đô thị, mỗi quỹ đạo 7–11 s, tốc độ p99 khoảng 17.8 m/s, phanh mạnh nhất 3.6 m/s². Ngay ở cruise 20 m/s chưa thấy, PercMLP+ đã đâm 0.4–1.5 % lưới, cho thấy ngoài phạm vi dữ liệu thì không có bảo đảm. (4) **Thống kê sự kiện hiếm**: với 2883 quỹ đạo test, độ chính xác tốt nhất cỡ 0.1 %, còn xa mức tỷ lệ tai nạn của người lái thật. Bước tiếp theo: chạy chế độ bóng trên xe thật để so quyết định với IDM, kiểm thử phần cứng trong vòng lặp với cảm biến thật, dùng tác nhân phản ứng học từ dữ liệu thay vì IDM, và mở rộng dữ liệu sang đường cao tốc, cắt vào thật và thời tiết xấu.
