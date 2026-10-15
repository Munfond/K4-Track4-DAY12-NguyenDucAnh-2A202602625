# REPORT — Lab Day 12

> Kết quả được chạy từ mã hoàn thiện với MSSV `2A202602625`, ba seed huấn luyện và dữ liệu đầy đủ. Số liệu bên dưới lấy từ `evidence/*.json`.

| | |
|---|---|
| Họ và tên | Nguyễn Đức Anh |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602625` |
| GitHub | [Munfond](https://github.com/Munfond/K4-Track4-DAY12-NguyenDucAnh-2A202602625) |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `9574` / `1410, 1411, 1412` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `3` |

Lệnh tái lập: `python run_all.py --student-id 2A202602625 --seeds 3` (không dùng `--quick`).

Trên Google Colab, chạy các ô sau theo thứ tự trong một runtime mới (CPU):

```python
!git clone https://github.com/Munfond/K4-Track4-DAY12-NguyenDucAnh-2A202602625.git /content/lab12
%cd /content/lab12
!pip -q install -r requirements.txt
%env OPENBLAS_NUM_THREADS=1
%env OMP_NUM_THREADS=1
%env MKL_NUM_THREADS=1
!python -m unittest discover -s tests -v
!python run_all.py --student-id 2A202602625 --seeds 3
!python scripts/check_submission.py --save-verify
```

Sau khi chạy lại trên Colab, tải `evidence/` về trước khi kết thúc phiên; nếu số làm tròn thay đổi theo môi trường thư viện, đối chiếu và cập nhật báo cáo theo bằng chứng vừa sinh. Các kiểm tra ở bản nộp này được chạy trên CPU tại máy cục bộ, chưa chạy trong phiên Colab thật.

## Kết quả cuối

Nguồn: `evidence/closedloop.json`, `openloop.json`, `gate.json`; dải L2 của PercMLP+ được tính lại từ các model đã lưu và ghi ở `evidence/report_checks.json`. Với mô hình học, ghi **min–max qua ba seed**, không dùng một seed đại diện.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,264810 | 20,0486 | BLOCK: R2, R3, R4, R6 |
| IDM | 1,548346 | 0,0694 | (baseline) |
| EgoMLP | 0,763645–0,770505 | 13,1807–32,7090 | BLOCK: R2, R3, R4, R5, R6 |
| PercMLP | 0,621927–0,629180 | 0,4856–1,0059 | BLOCK: R2, R3, R4, R6 |
| PercMLP+ (sau khi sửa) | 0,788137–0,816653 | 0,2081–0,2428 | RELEASE: qua cả 7 quy tắc của lab |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Theo [data_audit.json](evidence/data_audit.json), tương quan của `Acc_LV` với đạo hàm tốc độ xe dẫn chỉ là **0,045354**, trong khi xe theo sau đạt **0,929087**; do đó không dùng trực tiếp `Acc_LV` để tìm ca phanh.
Tôi dùng `a_lead` được tính bằng sai phân trung tâm từ `Speed_LV` đã làm mượt bằng trung bình trượt, thay vì coi nhiễu một khung là sự kiện phanh.
Dữ liệu có **9.993** quỹ đạo, loại **383** bản trùng còn **9.610**; nếu chia trước khi loại trùng, cùng một hành vi có thể xuất hiện ở cả train và test dù khác ID.
Tôi chia theo quỹ đạo với seed **9574**, vì các khung kề nhau có chung lịch sử và tương lai; chia ngẫu nhiên theo khung sẽ gây rò rỉ và làm điểm kiểm thử lạc quan.
Ba vấn đề cần xử lý là gia tốc xe dẫn không đáng tin, quỹ đạo trùng, và phụ thuộc giữa các khung trong cùng quỹ đạo; loại trùng trước rồi mới chia giúp bộ kiểm thử độc lập hơn.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Theo [replay.json](evidence/replay.json), chạy lại cùng mã có **0/600** đoạn khác; refactor float32 có **600/600** đoạn không bằng chính xác nhưng **0** đoạn vượt dung sai, sai khác lớn nhất **0,000403132 m/s²**.
Dùng điều kiện `|new-golden| <= 0,001 + 0,001*|golden|` theo bộ khung: sai khác do độ chính xác và thứ tự phép tính vẫn nằm trong mức chấp nhận cho lệnh gia tốc. Dung sai này phù hợp kiểm tra refactor của lab; sản phẩm thật phải xác nhận bằng yêu cầu điều khiển.
AEB làm lệch **4/600** đoạn, chỉ số **89, 146, 212, 588**. Khi mở đoạn **89**, khung **82** có gap **6,1588 m**, closing **4,1994 m/s**, TTC xấp xỉ **1,46659 s**, khiến lệnh từ **−4,56443** đổi thành **−6 m/s²**; chi tiết được sinh lại trong [report_checks.json](evidence/report_checks.json).
Lệch xuất hiện khi TTC dưới ngưỡng **1,5 s** và lệnh cũ chưa phanh mạnh bằng AEB; TTC thấp một mình chưa đủ tạo sai khác nếu IDM đã yêu cầu phanh mạnh hơn.
Phát nhanh nhất có **0** khung trễ, **0** khung rớt; phát theo thời gian thật có **582** khung trễ và **575** khung rớt trên **59.694** khung, làm **373/600** đoạn vượt dung sai so với golden.
Ở chế độ nhanh nhất không có áp lực deadline. Chế độ thời gian thật giữ thời điểm đến của thông điệp, nên xử lý đột biến có thể quá hạn, chặn khung sau và buộc giữ lệnh cũ; đây là chi phí xử lý mô phỏng, không phải đo latency thật của phần cứng.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Theo [scenarios.json](evidence/scenarios.json), tốc độ cruise là **12 m/s**, khoảng cách **2,760–59,312 m**, tốc độ tiến gần **0–12 m/s**; lưới **60 × 45 = 2.700 ô**, kèm **614** sự kiện phanh khai thác từ log.
Biên bất khả thi là `gap < dv*t_react + dv²/(2*b_max)`: xe mất khoảng cách trong lúc chờ phản ứng, rồi cần thêm quãng đường để triệt tiêu tốc độ tiến gần; với `dv=0` không có nguy cơ do tiến gần.
Khi trễ từ **0 lên 0,6 s**, phần bất khả thi và phần va chạm của IDM cùng tăng từ **2,222% lên 7,444%**, tức **5,222 điểm phần trăm**; phần tránh được nhưng IDM vẫn đâm bằng **0%** ở cả hai độ trễ.
Kiểm chứng bằng bộ phanh cực đại cho **0 ô lệch** xa biên tại cả hai độ trễ; vì thế cần tách va chạm do điều kiện vật lý khỏi thất bại có thể quy cho bộ điều khiển.
Ngưỡng phanh người lái ở phân vị **99,9%** là **3,625 m/s²**, trong khi **8,556%** lưới đòi hỏi lớn hơn mức đó và cơ cấu chấp hành cho phép **9 m/s²**. Đây là phân vị mạnh, không phải cực đại tuyệt đối.
Các ca khai thác từ log không phủ đều vùng khoảng cách nhỏ và tốc độ tiến gần lớn; điểm tốt trên log không chứng minh mô hình đã học cách xử lý vùng phanh gấp của lưới.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Theo [openloop.json](evidence/openloop.json) và [closedloop.json](evidence/closedloop.json), bảng dưới dùng L2 trung bình và tỷ lệ va chạm **trung bình qua seed** để khớp chính xác cách xếp hạng của pipeline; bảng kết quả cuối ghi thêm dải seed.

| Bộ điều khiển | L2 trung bình (m) | Hạng L2 | Va chạm trung bình (%) | Hạng vòng kín |
|---|---:|---:|---:|---:|
| constant-velocity | 1,264810 | 3 | 20,0486 | 3 |
| IDM | 1,548346 | 4 | 0,0694 | 1 |
| EgoMLP | 0,766740 | 2 | 22,1182 | 4 |
| PercMLP | 0,625702 | 1 | 0,7747 | 2 |

Bốn cặp đảo thứ tự là **constant-velocity/IDM**, **constant-velocity/EgoMLP**, **IDM/EgoMLP**, **IDM/PercMLP**; L2 thấp trên trạng thái người lái không đảm bảo va chạm thấp khi mô hình tự điều khiển.
IDM có L2 cao vì dự báo xe dẫn giữ nguyên tốc độ và hành vi điều khiển không bắt chước chính xác người lái, nhưng phản hồi theo khoảng cách và tốc độ tiến gần giúp nó duy trì an toàn trong thế giới mô phỏng.
EgoMLP chỉ thấy lịch sử tốc độ bản thân, thiếu khoảng cách và tốc độ xe dẫn; cùng một lịch sử có thể dẫn tới quyết định tăng hoặc giảm tốc khác nhau. Sai số nhỏ làm trạng thái tự lái lệch dần khỏi log, nên L2 tốt hơn constant-velocity vẫn có thể đâm nhiều hơn.
Lát cắt sắp phanh chỉ chiếm **4,323%** cửa sổ: L2 của PercMLP trên lát cắt là **0,891 m**, cao hơn **0,626 m** toàn bộ; trung bình che những thời điểm cần phản ứng nhất.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Theo mục `logs.EgoMLP.per_seed` của [closedloop.json](evidence/closedloop.json), trên cùng **2.883** quỹ đạo test, seed **1410** có **943** va chạm (**32,709%**), seed **1411** có **380** (**13,181%**), seed **1412** có **590** (**20,465%**).
Dải rộng **19,528 điểm phần trăm**, dù L2 của ba lần huấn luyện chỉ nằm khoảng **0,764–0,771 m**; thay đổi ngẫu nhiên trong học có thể tạo ra phản hồi vòng kín rất khác nhau.
Khoảng Wilson **95%** của seed tốt nhất là **[11,995%; 14,465%]**, của seed xấu nhất là **[31,020%; 34,444%]**; hai khoảng không chồng nhau nên sự chênh lệch lớn hơn mức bất định nhị thức thể hiện trong từng lần chạy.
Nếu chỉ lấy seed **1411**, tôi có thể báo EgoMLP va chạm ít hơn constant-velocity (**20,049%**); lấy seed **1410** lại dẫn tới thứ tự ngược. Chọn seed đẹp nhất sẽ làm kết luận phát hành thiếu tin cậy.
Wilson mô tả bất định của tỷ lệ trên tập thử theo giả định lấy mẫu, không bao trùm biến thiên do huấn luyện hay khoảng cách mô phỏng–thực tế. Tôi báo cả dải qua seed, khoảng tin cậy và dùng seed xấu nhất tại cổng.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Theo [agents.json](evidence/agents.json), **T=1,4 s** cho khoảng cách Wasserstein giữa các phân bố time headway nhỏ nhất trong các giá trị thử: **0,348861**, so với **0,872318** của tác nhân không phản ứng; median headway người lái là **1,876698 s**.
Đây là khớp một phân bố, chưa chứng minh xe sau có mọi hành vi như người thật. Xe sau bắt đầu cách **18,8 m**, phản ứng trễ **0,6 s**; bảng dưới dùng lưới với độ trễ xe thử **0,6 s**, hàng mô hình học được chọn theo seed có `reactive.any` xấu nhất như pipeline.

| Bộ điều khiển | Bất kỳ va chạm: xe sau phát lại (%) | Bất kỳ va chạm: xe sau phản ứng (%) | Va chạm phía sau: phát lại / phản ứng (%) |
|---|---:|---:|---:|
| constant-velocity | 77,630 | 77,630 | 0 / 0 |
| IDM | 75,741 | 7,444 | 68,296 / 0 |
| EgoMLP | 78,222 | 78,222 | 0 / 0 |
| PercMLP | 83,407 | 33,963 | 49,741 / 0 |

Xe sau phát lại giữ tốc độ dù xe thử vừa phanh, nên tính **68,296%** va chạm sau cho IDM; gọi là “giả” theo nghĩa phát sinh từ giả định tác nhân không thích nghi, không phải bằng chứng IDM phanh sai trong thế giới có người lái phản ứng.
Với phát lại, IDM (**75,741%**) chỉ nhỉnh hơn constant-velocity (**77,630%**) một ít; khi xe sau phản ứng, chênh lệch thành **7,444% so với 77,630%**. Trong bộ số này hai bộ không đảo hạng, nhưng mức độ lợi ích của IDM đổi rất lớn.
PercMLP từ **83,407%** xuống **33,963%**, qua đó đổi hạng so với constant-velocity. Kết luận phụ thuộc mô hình tương tác; tác nhân không phản ứng có thể phạt chính hành vi phanh cần thiết để tránh xe phía trước.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Theo [gate.json](evidence/gate.json), PercMLP rớt **R2, R3, R4, R6**: cận trên Wilson trên log **1,441% > 1,069%** cho phép; trên lát cắt phanh cận trên **9,2% > 5,556%**; phần lưới tránh được nhưng đâm **26,519% > 2%**; thế giới có xe sau phản ứng **33,963% > 9,444%**. R1, R5, R7 đều qua; constant-velocity rớt cùng bốn quy tắc, EgoMLP còn rớt thêm R5.
Hai nguyên nhân thể hiện ở các phép thử khác nhau: thiếu vùng cắt vào/phanh mạnh trong log liên quan tới **R4/R6**, còn sai số bắt chước tích luỹ khi tự lái liên quan tới **R2/R3**. Đây là chẩn đoán phù hợp bằng chứng; do hai biện pháp sửa được áp dụng cùng nhau, thí nghiệm này chưa tách riêng đóng góp nhân quả của mỗi biện pháp.
Theo [fix.json](evidence/fix.json), tôi bổ sung cửa sổ từ **20.000** kịch bản cắt vào do IDM gắn nhãn, rồi làm **2** vòng DAgger mỗi seed. DAgger thu các trạng thái do chính mô hình tạo ra trên **log huấn luyện**, dùng dự báo IDM tại đúng các trạng thái đó làm nhãn; không lấy trạng thái test vào huấn luyện.
Sau sửa, **R2/R3/R4/R6 đều chuyển sang qua**: seed xấu nhất có **7/2.883** va chạm (**0,243%**, Wilson trên **0,500%**), lát cắt phanh **2/180** (**1,111%**, cận trên **3,960%**); R4 còn **0,148%**, R6 còn **7,593%**. Tất cả bảy quy tắc qua, và `decision_PercMLP+` trong `fix.json` khớp cổng.
L2 trung bình tăng từ **0,625702 lên 0,806811 m**, trong khi dải va chạm giảm từ **0,4856–1,0059%** xuống **0,2081–0,2428%**. Tối ưu độ giống người lái trên log và khả năng phục hồi vòng kín không đồng nhất; cần đánh giá cả hai.
PercMLP+ học từ IDM và chỉ đạt tiêu chí không-kém-hơn theo biên đã chọn; nó vẫn va chạm nhiều hơn IDM (**0,0694%**) trên log này. Với tốc độ ngoài huấn luyện **20 m/s**, vùng tránh được nhưng đâm là **0,926–2,259%** so với IDM **0%**; đây là lý do chưa thể kết luận mô hình vượt thầy.
Tôi chấp nhận trạng thái **RELEASE trong phạm vi cổng của lab**, nhưng chưa thay IDM trong sản phẩm thật. Trước quyết định triển khai cần tập kiểm thử độc lập ngoài phân bố, thử riêng từng biện pháp sửa, thêm seed, cảm nhận/thời gian thực và tác nhân đa dạng, rồi đánh giá trên đường thử kín và chạy shadow có giám sát.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

**Giới hạn hình học và động lực học:** mô phỏng chỉ theo chiều dọc; kết quả trên lưới cruise **12 m/s**, dù có **2.700 ô**, không kiểm chứng chuyển làn, đường cong, người đi bộ, mặt đường hay sai số cơ cấu chấp hành. Cần kịch bản đa tác nhân có chuyển động ngang, thay đổi độ bám và kiểm thử trên đường thử kín.
**Giới hạn cảm nhận và thời gian:** [closedloop.json](evidence/closedloop.json) chỉ thử nhiễu gap **1 m**, tốc độ xe dẫn **0,5 m/s**; [replay.json](evidence/replay.json) mô phỏng **582** khung trễ và **575** khung rớt. Chưa có pipeline cảm biến thật, che khuất hay phân bố độ trễ của phần cứng; cần đo và phát lại cảm biến, kiểm thử lỗi mất mục tiêu, rồi kiểm thử thời gian thực trên phần cứng.
**Giới hạn hành vi của thế giới:** xe dẫn từ log không phản ứng khi xe thử đổi cách lái; xe sau hiệu chỉnh **T=1,4 s** chỉ khớp một phân bố với W1 **0,348861** trong [agents.json](evidence/agents.json). Cần nhiều mô hình lái xe có mức hung hăng và khả năng phản ứng khác nhau, kiểm chứng trên dữ liệu tương tác độc lập.
**Giới hạn dữ liệu và suy rộng:** **9.610** quỹ đạo sau loại trùng vẫn là một nguồn dữ liệu ngắn; chỉ **614** sự kiện phanh và **8,556%** lưới đòi hỏi phanh mạnh hơn phân vị người lái đã chọn. Cần kiểm thử ngoài phân bố theo địa điểm, tốc độ, thời tiết, tình huống hiếm và dữ liệu dài hơn, tránh đồng nhất số ô nhiều với độ phủ thế giới thật.
**Giới hạn bằng chứng thống kê và mục tiêu:** **3** seed trên cùng tập test chưa bao trùm mọi cách huấn luyện; Wilson không bảo đảm an toàn triển khai, và cổng chủ yếu xét va chạm nên chưa chứng minh độ êm hay hiệu quả giao thông. Tôi chỉ xem kết quả là căn cứ tiếp tục thử nghiệm, bổ sung tiêu chí và đánh giá độc lập trước khi cân nhắc phát hành.
