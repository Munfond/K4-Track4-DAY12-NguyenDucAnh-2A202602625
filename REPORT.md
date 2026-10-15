# REPORT — Lab Day 12

| Thông tin | Giá trị |
|---|---|
| Họ và tên | Nguyễn Đức Anh |
| MSSV | `2A202602625` |
| GitHub | [Munfond](https://github.com/Munfond/K4-Track4-DAY12-NguyenDucAnh-2A202602625) |
| Seed chia dữ liệu | `9574` |
| Seed mô hình | `1410, 1411, 1412, 1413, 1414` |
| Số seed huấn luyện | `5` |
| Môi trường | Local, CPU; dữ liệu đầy đủ, không `--quick` |
| Phiên bản thư viện | Python 3.12.14; NumPy 2.5.4; pandas 3.0.6; matplotlib 3.11.2 |

Lệnh chạy cuối: `python run_all.py --student-id 2A202602625` (mặc định **5 seed**).
Trên máy này, dùng Python trong môi trường `work/.venv` để không phụ thuộc cấu hình lệnh `python` toàn hệ thống:

```powershell
$env:PYTHONUTF8 = "1"
$env:OPENBLAS_NUM_THREADS = "1"
$env:OMP_NUM_THREADS = "1"
$env:MKL_NUM_THREADS = "1"
& "./work/.venv/Scripts/python.exe" run_all.py --student-id 2A202602625
& "./work/.venv/Scripts/python.exe" scripts/check_submission.py --save-verify
```

Nguồn số liệu là các JSON trong `evidence/`; kết quả self-check được lưu ở [verify.txt](evidence/verify.txt).

## Kết quả cuối

L2 là trung bình sai số vị trí tại 1, 2, 3 giây. Mô hình học ghi **min–max qua cả 5 seed**, hai baseline xác định ghi một giá trị.
Nguồn: [openloop.json](evidence/openloop.json), [closedloop.json](evidence/closedloop.json), [gate.json](evidence/gate.json); dải L2 PercMLP+ được tính lại từ checkpoint và lưu ở [report_checks.json](evidence/report_checks.json).
Tỷ lệ trên lưới dùng toàn bộ số ô làm mẫu số, kể cả khi chỉ đếm va chạm thuộc vùng tránh được; không diễn giải thành tỷ lệ có điều kiện trong riêng vùng tránh được.
Cổng R2/R3 dùng entry của seed có tỷ lệ va chạm toàn log cao nhất, đúng cách chọn của pipeline; bảng tác nhân phản ứng chọn seed theo `reactive.any` xấu nhất.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---:|---:|---|
| constant-velocity | 1,264810 | 20,0486 | BLOCK: R2, R3, R4, R6 |
| IDM | 1,548346 | 0,0694 | Baseline |
| EgoMLP | 0,763645–0,770505 | 13,1807–32,7090 | BLOCK: R2, R3, R4, R5, R6 |
| PercMLP | 0,621927–0,629180 | 0,4856–1,0059 | BLOCK: R2, R3, R4, R6 |
| PercMLP+ | 0,788137–0,835501 | 0,2081–0,2775 | RELEASE: qua 7/7 |

## Câu 1 · Dữ liệu

Theo [data_audit.json](evidence/data_audit.json), tương quan `Acc_LV` với đạo hàm tốc độ xe dẫn chỉ **0,045354**, so với **0,929087** ở xe theo sau; cột gia tốc xe dẫn không đủ tin cậy để khai thác ca phanh.
Tôi dùng `a_lead`: sai phân trung tâm từ `Speed_LV` đã làm mượt bằng trung bình trượt, để giảm ảnh hưởng của nhiễu một khung.
Có **9.993** quỹ đạo gốc; loại **383** bản trùng còn **9.610**. Phải loại trùng trước khi chia, vì khác ID vẫn có thể là cùng một log.
Nếu một bản trùng vào train và một bản vào test, mô hình đã thấy chính hành vi được chấm, làm kết quả kiểm thử lạc quan giả tạo.
Seed chia **9574** tạo **6.727** quỹ đạo train và **2.883** test; chia theo quỹ đạo tránh đưa các khung gần nhau, chung lịch sử và tương lai, sang hai tập.
Ba vấn đề chính là gia tốc xe dẫn không đáng tin, quỹ đạo trùng và rò rỉ do phụ thuộc giữa các khung; kiểm tra cả ba là điều kiện trước khi tin điểm mô hình.

## Câu 2 · Phát lại

Theo [replay.json](evidence/replay.json), chạy lại cùng mã có **0/600** đoạn khác; refactor float32 khác chính xác ở **600** đoạn nhưng **0** đoạn vượt dung sai, sai khác lớn nhất **0,000403132 m/s²**.
Tiêu chí `|new − golden| ≤ 0,001 + 0,001 × |golden|` chấp nhận sai số do độ chính xác và thứ tự phép tính; hai NaN ở cùng vị trí đệm được coi là bằng nhau. Dung sai sản phẩm thật cần được xác nhận bằng yêu cầu điều khiển.
AEB làm lệch **4/600** đoạn: **89, 146, 212, 588**; đây là thay đổi hành vi có chủ đích, khác với sai số float32.
[report_checks.json](evidence/report_checks.json) ghi đoạn **89**, khung **82**: gap **6,1588 m**, closing **4,1994 m/s**, TTC **1,46659 s**; TTC dưới **1,5 s** khiến lệnh **-4,56443** đổi thành **-6,0 m/s²**. Nếu IDM đã phanh mạnh hơn AEB thì TTC thấp vẫn không làm lệnh đổi.
Phát nhanh nhất có **0** khung trễ và **0** khung rớt; phát thời gian thật có **582** trễ, **575** rớt trên **59.694** khung, làm **373** đoạn vượt dung sai.
Phát nhanh nhất không áp deadline; phát thời gian thật giữ thời điểm đến nên chi phí xử lý đột biến làm quá hạn, chặn khung sau và buộc giữ lệnh cũ. Đây là mô phỏng chi phí xử lý, chưa phải đo latency trên phần cứng thật.

## Câu 3 · Kịch bản và biên bất khả thi

Theo [scenarios.json](evidence/scenarios.json), cruise **12 m/s**, gap **2,760–59,312 m**, closing **0–12 m/s**; lưới **60 × 45 = 2.700** ô, kèm **614** sự kiện phanh từ log.
Biên là `gap < dv × t_react + dv² / (2 × b_max)`: khoảng cách mất trong lúc chờ phản ứng cộng quãng đường cần để triệt tiêu tốc độ tiến gần; khi `dv=0` không có nguy cơ do tiến gần.
Độ trễ **0 → 0,6 s** làm phần bất khả thi tăng **2,222% → 7,444%**, tức **5,222 điểm phần trăm**; phanh cực đại có **0** và **0** ô lệch xa biên.
Vùng đâm IDM cũng tăng **2,222% → 7,444%**, còn vùng tránh được nhưng IDM vẫn đâm là **0%** ở cả hai độ trễ; vì vậy không quy toàn bộ va chạm cho lỗi điều khiển.
Phân vị **99,9%** độ lớn phanh người lái là **3,625 m/s²**; **8,556%** lưới đòi hỏi mạnh hơn. Đây là phân vị, không phải cực đại tuyệt đối của mọi khung.
Các ca log không phủ đều vùng gap nhỏ và closing lớn; lưới có tham số làm lộ vùng thiếu dữ liệu, còn biên vật lý tách điều kiện ai cũng đâm khỏi thất bại mà bộ điều khiển có thể tránh.

## Câu 4 · Vòng hở và vòng kín

| Bộ điều khiển | L2 trung bình (m) | Hạng vòng hở | Va chạm trung bình (%) | Hạng vòng kín |
|---|---:|---:|---:|---:|
| constant-velocity | 1,264810 | 3 | 20,0486 | 3 |
| IDM | 1,548346 | 4 | 0,0694 | 1 |
| EgoMLP | 0,766459 | 2 | 21,1585 | 4 |
| PercMLP | 0,625642 | 1 | 0,7076 | 2 |

Bảng dùng [openloop.json](evidence/openloop.json) và [closedloop.json](evidence/closedloop.json): trung bình qua seed để khớp cách xếp hạng của pipeline; bảng kết quả cuối phía trên ghi min–max.
Có **4** cặp đảo thứ tự: **constant-velocity/IDM**; **constant-velocity/EgoMLP**; **IDM/EgoMLP**; **IDM/PercMLP**; L2 thấp trên trạng thái người lái không bảo đảm an toàn khi bộ điều khiển tự tạo trạng thái mới.
IDM dự báo xe dẫn giữ tốc độ và không bắt chước chính xác người lái nên L2 cao, nhưng dùng phản hồi khoảng cách và tốc độ tiến gần để điều chỉnh phanh liên tục, giúp tỷ lệ va chạm thấp.
EgoMLP chỉ thấy lịch sử tốc độ bản thân, thiếu gap và tốc độ xe dẫn; cùng một lịch sử có thể cần quyết định khác nhau. Sai số nhỏ tích luỹ khiến nó rời khỏi phân bố log và có thể đâm nhiều hơn constant-velocity.
Lát cắt sắp phanh chỉ chiếm **4,323%** cửa sổ; L2 PercMLP trên lát cắt **0,894 m**, cao hơn trung bình **0,626 m**.
Do đó phải đọc cả trung bình, lát cắt khó và kết quả tự lái; một điểm dự báo đẹp có thể che đúng các thời điểm an toàn quan trọng nhất.

## Câu 5 · Biến thiên qua seed và khoảng tin cậy

Trên **2.883** quỹ đạo test, [closedloop.json](evidence/closedloop.json) ghi EgoMLP qua **5** seed: `1410`: 943 ca (32,709%); `1411`: 380 ca (13,181%); `1412`: 590 ca (20,465%); `1413`: 492 ca (17,066%); `1414`: 645 ca (22,373%).
Dải va chạm là **13,181–32,709%**, rộng **19,528 điểm phần trăm**, trong khi dải L2 chỉ **0,764–0,771 m**.
Wilson **95%** của seed tốt nhất **1411** là **[11,995%; 14,465%]**, của seed xấu nhất **1410** là **[31,020%; 34,444%]**; hai khoảng không chồng nhau.
Chỉ báo seed **1411** sẽ cho EgoMLP ít đâm hơn constant-velocity (**20,049%**), trong khi seed **1410** cho kết luận ngược; chọn một lần chạy đẹp dễ dẫn tới quyết định phát hành sai.
Wilson mô tả bất định lấy mẫu của tỷ lệ trong từng lần chạy theo giả định nhị thức; nó không bao trùm biến thiên huấn luyện, phụ thuộc giữa các tình huống hay sai khác mô phỏng–thực tế.
Tôi báo min–max và khoảng tin cậy, giữ nguyên tập test, đồng thời dùng seed xấu nhất tại cổng; nhiều seed cung cấp bằng chứng về độ ổn định, chưa phải bảo đảm an toàn tuyệt đối.

## Câu 6 · Tác nhân phản ứng

| Bộ điều khiển | Xe sau phát lại: mọi va chạm (%) | Xe sau phản ứng: mọi va chạm (%) | Va chạm sau: phát lại / phản ứng (%) |
|---|---:|---:|---:|
| constant-velocity | 77,630 | 77,630 | 0,000 / 0,000 |
| IDM | 75,741 | 7,444 | 68,296 / 0,000 |
| EgoMLP | 78,222 | 78,222 | 0,000 / 0,000 |
| PercMLP | 83,407 | 33,963 | 49,741 / 0,000 |
| PercMLP+ | 75,481 | 7,704 | 67,778 / 0,000 |

Theo [agents.json](evidence/agents.json), **T=1,4 s** cho Wasserstein nhỏ nhất trong các T đã thử: **0,348861**, tốt hơn **0,872318** của tác nhân không phản ứng; median headway người lái **1,876698 s**.
Bảng dùng độ trễ xe thử **0,6 s**, gap xe sau **18,8 m** và trễ xe sau **0,6 s**; mỗi hàng mô hình học lấy seed có `reactive.any` xấu nhất như pipeline.
Xe sau phát lại không biết xe thử vừa phanh, tạo **68,296%** va chạm sau cho IDM, còn khi phản ứng là **0,000%**; đây là “va chạm giả” do giả định thế giới thiếu thích nghi, không đủ để kết luận IDM phanh sai.
IDM so với constant-velocity là **75,741% / 77,630%** khi phát lại và **7,444% / 77,630%** khi phản ứng: hạng giữ nguyên nhưng mức lợi ích đổi rất lớn.
PercMLP từ **83,407%** xuống **33,963%**, đổi hạng so với constant-velocity; lựa chọn tác nhân có thể đổi cả kết luận về bộ điều khiển.
Hiệu chỉnh headway chỉ khớp một phân bố, chưa xác nhận mọi hành vi người thật; cần thử thêm nhiều tác nhân và mức phản ứng để tránh đặt niềm tin vào một thế giới mô phỏng duy nhất.

## Câu 7 · Cổng phát hành và sửa

| Quy tắc / số đo dùng để xét | PercMLP | PercMLP+ | Giới hạn chấp nhận |
|---|---:|---:|---:|
| R2: cận trên Wilson trên log (%) | 1,441 | 0,547 | ≤ 1,069 |
| R3: cận trên Wilson lát cắt phanh (%) | 9,228 | 3,960 | ≤ 5,556 |
| R4: ô tránh được nhưng đâm / toàn lưới (%) | 26,519 | 0,259 | ≤ 2,000 |
| R6: mọi va chạm với xe sau phản ứng (%) | 33,963 | 7,704 | ≤ 9,444 |

Theo [gate.json](evidence/gate.json), PercMLP bị chặn bởi **R2, R3, R4, R6**; constant-velocity rớt **R2, R3, R4, R6**, EgoMLP rớt **R2, R3, R4, R5, R6**. Bảng ghi đúng số dùng tại cổng, không thay ngưỡng.
R2/R3 phù hợp với sai số bắt chước tích luỹ khi tự lái; R4/R6 phơi bày vùng cắt vào/phanh mạnh mà log ít phủ. Đây là hai nguyên nhân cần sửa ở trạng thái tự tạo và ở họ kịch bản thiếu dữ liệu.
Theo [fix.json](evidence/fix.json), thêm dữ liệu từ **20.000** kịch bản cắt vào và **2** vòng DAgger mỗi seed: thu trạng thái mô hình tự đi vào trên log train rồi gắn nhãn dự báo IDM tại chính trạng thái ấy; không huấn luyện trên test.
Sau sửa, **R2/R3/R4/R6 đều qua**, tổng cộng **7/7** quy tắc; seed xấu nhất có **8/2.883** va chạm, lát cắt phanh **2/180**; `fix.json` và `gate.json` cùng quyết định RELEASE.
L2 trung bình từ **0,625642 lên 0,814207 m**, nhưng va chạm từ **0,4856–1,0059%** xuống **0,2081–0,2775%**; điểm giống log và chất lượng tự lái không đồng nhất.
PercMLP+ học từ IDM, chưa chứng minh tốt hơn thầy: IDM trên log chỉ **0,0694%** va chạm; trên lưới cruise **20 m/s** ngoài dải cruise của các ca cắt vào tổng hợp, PercMLP+ còn **0,037–2,259%** vùng tránh được nhưng đâm, IDM **0%**.
Qua cổng chỉ đạt tiêu chí không kém hơn theo các biên của lab, chưa đủ để thay IDM ngoài đời; cần tập độc lập, thử tách từng biện pháp sửa, thêm tình huống ngoài phân bố và kiểm thử phần cứng/đường thử kín trước khi quyết định triển khai.

## Câu 8 · Giới hạn và quyết định triển khai

**Hình học và động lực học:** lưới **2.700** ô ở **12 m/s** vẫn chỉ mô phỏng chiều dọc, chưa kiểm chứng chuyển làn, đường cong, người đi bộ hay độ bám; cần kịch bản ngang và biến thiên động lực học trên đường thử kín.
**Cảm nhận và thời gian:** [closedloop.json](evidence/closedloop.json) chỉ thêm nhiễu có độ lệch chuẩn gap **1 m**, tốc độ **0,5 m/s**; [replay.json](evidence/replay.json) mô phỏng **582** khung trễ và **575** khung rớt. Cần cảm biến, che khuất/mất mục tiêu và latency đo trên phần cứng thật.
**Tương tác:** xe dẫn phát lại không phản ứng khi xe thử đổi hành vi; xe sau với **T=1,4 s**, W1 **0,348861** trong [agents.json](evidence/agents.json) chỉ khớp một đặc trưng. Cần nhiều kiểu lái và kiểm chứng trên dữ liệu tương tác độc lập.
**Độ phủ dữ liệu:** **9.610** quỹ đạo sạch, **614** sự kiện phanh và **8,556%** lưới đòi phanh mạnh hơn phân vị người lái chưa đại diện mọi địa điểm, thời tiết và sự kiện hiếm; phải mở rộng tập thử ngoài phân bố.
**Thống kê và mục tiêu:** **5** seed dùng chung tập test không bao quát mọi lần huấn luyện; Wilson không đo khoảng cách mô phỏng–thực tế. Cổng tập trung va chạm, chưa đủ chứng minh độ êm, hiệu quả giao thông và hành vi khi hệ thống lỗi.
Vì các giới hạn này, tôi dùng kết quả để cho phép bước thử nghiệm tiếp theo có giám sát, chưa coi là chứng nhận an toàn sản phẩm; cần đánh giá độc lập và vận hành shadow trước khi cân nhắc phát hành thực tế.

## Phụ lục · Tái tạo bằng chứng bổ sung

<details>
<summary>Mã sinh report_checks.json từ log và checkpoint sau khi chạy pipeline</summary>

Lưu đoạn sau thành `work/report_checks.py`, rồi chạy `python work/report_checks.py` từ thư mục repo. File JSON này chứa khung AEB đã mở và L2 từng seed của PercMLP+, không sửa số bằng tay.

```python
"""Generate supplementary report evidence from the unmodified lab pipeline."""
import json
import os
import platform
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))
import numpy as np
import pandas as pd
import matplotlib
from lab12 import config as C, replay as R, models
from lab12.steps import Ctx

closed = json.loads((C.EVID_DIR / "closedloop.json").read_text())
ctx = Ctx("2A202602625", n_seeds=len(closed["logs"]["EgoMLP"]["per_seed"]))
segs = R.read_log(C.WORK_DIR / "replay_log.jsonl")
golden, _ = R.replay(segs, lambda: R.IDMStream())
aeb, _ = R.replay(segs, lambda: R.IDMStream(aeb_ttc=1.5))
cmp = R.compare_with_tol(golden, aeb)
examples = []
for seg in cmp["differing_segments"]:
    can = [m for m in segs[seg] if m["topic"] == "/can"]
    perception = [m for m in segs[seg] if m["topic"] == "/perception"]
    changed = np.flatnonzero(~np.isclose(aeb[seg], golden[seg], atol=1e-3, rtol=1e-3, equal_nan=True))
    for k in changed:
        gap = perception[k]["gap"]
        closing = can[k]["v_ego"] - perception[k]["v_lead"]
        examples.append({"segment": int(seg), "frame": int(k), "gap_m": gap,
                         "closing_m_s": closing, "ttc_s": gap / closing,
                         "golden_cmd": float(golden[seg, k]), "aeb_cmd": float(aeb[seg, k])})
Xe, Xp, Y, brake, _ = models.build_windows(ctx.logs, ctx.test)
fixed_l2 = []
for seed in ctx.seeds:
    m = models.Learned.load(C.WORK_DIR / "models" / f"percfix_{seed}.npz")
    fixed_l2.append({"seed": seed, **models.l2_summary(m.predict(Xp), Y, brake)})
out = {"student": ctx.sp, "n_seeds": len(ctx.seeds), "replay_aeb_changed_frames": examples,
       "PercMLP+_openloop_per_seed": fixed_l2,
       "environment": {"python": platform.python_version(), "numpy": np.__version__,
                       "pandas": pd.__version__, "matplotlib": matplotlib.__version__,
                       "system": platform.system(),
                       "thread_settings": {key: os.environ.get(key) for key in
                                           ("OPENBLAS_NUM_THREADS", "OMP_NUM_THREADS", "MKL_NUM_THREADS")}}}
(C.EVID_DIR / "report_checks.json").write_text(json.dumps(out, indent=2))
print(json.dumps(out, indent=2))
```

</details>
