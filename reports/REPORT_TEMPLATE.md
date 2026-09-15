# Báo cáo Ngày 3 — Tracking Annotation

Họ tên / nhóm: **Phạm Văn Phóng** — mã học viên **2A202602064** (làm cá nhân)
Ngày: **15/09/2026**

---

## 1. Quá trình gán nhãn

clip_01: 190 frame · 960×540 · 12.5 fps

| Mục | Giá trị |
| --- | --- |
| Công cụ | CVAT |
| Thời gian gán `clip_02` (warm-up) |  |
| Thời gian gán `clip_01` |  |
| Số track đã vẽ trong `clip_01` | 8 |
| Số keyframe trung bình mỗi track |  |

Ba tình huống khó nhất khi gán clip này, và cách xử lý:

1. **Xe mới vào khung (frame 51–78, track 5; tương tự track 4/6/7/8):** xe còn nhỏ/mờ nên khó quyết đặt bbox đầu tiên ở frame nào. Xử lý: chỉ mở track khi xe chắc chắn là 4 bánh và đã vào khung ≥ nửa thân.
2. **Hai xe chồng lấn (frame 82–115, track 5 & 6):** khi hai xe cắt/che nhau, box dễ vẽ rộng ôm cả xe và ID dễ nhảy. Xử lý: giữ nguyên ID cả hai, siết box ôm sát phần pixel thấy được.
3. **Xe rời khung / cuối clip (frame 149–151 track 4, 169–171 track 8, 190 track 1):** khó quyết frame cuối còn giữ box. Xử lý: dừng track ở frame cuối xe còn thấy thật, không kéo thêm frame đuôi.

## 2. Tự kiểm và kiểm chéo

Ba lượt tua tự kiểm:

- **Lượt 1 (nhìn ID):** kiểm tra ID bám đúng xe qua đoạn chồng lấn track 5 & 6 (frame 82–115) — không có ID nhảy (bản cuối IDSW = 0).
- **Lượt 2 (frame đầu/cuối mỗi track):** phát hiện xu hướng mở track sớm và giữ box muộn ở các track 4–8. Đối chiếu với model cho thấy rõ: ở frame 51, 52, 54 nhãn của tôi có 2 box còn ReID chỉ có 1 — đúng đoạn track 5 tôi mở sớm.
- **Lượt 3 (frame giữa):** rà bbox ở đoạn bị che, tìm các box lỏng (IoU thấp) quanh track 5/6 để vẽ khít lại.

Kiểm chéo: **không có** — làm cá nhân, không ghép cặp. Toàn bộ kiểm tra là tự kiểm ba lượt ở trên; không có `reports/review_partner.md`.

## 3. Pre-gold lock và chấm trước/sau rework

| Evidence | Giá trị |
| --- | --- |
| SHA-256 từ `evidence/pre-gold/clip_01/manifest.json` |  |
| Thời điểm khóa |  |
| Số row / frame / track trước khi mở reference | 646 row / 190 frame / 8 track |

| | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Bản pre-gold | 0.768 | 0.743 | 0.796 | 0.869 | 0.929 | 0.848 | 0.856 | 80 | 7 | 0 |
| Sau rework | 0.768 | 0.743 | 0.796 | 0.869 | 0.929 | 0.848 | 0.856 | 80 | 7 | 0 |

Bản pre-gold đã qua cả 3 cổng ngay khi khóa, nên không rework — bản cuối trùng khớp bản pre-gold về mọi chỉ số.

Qua cổng (`IDF1 >= 0.80`, `MOTA >= 0.75`, `MOTP >= 0.70`): **có**

Vì không rework, không có dòng sửa lỗi. Các vấn đề còn lại (được giữ nguyên trong bản cuối vì vẫn qua cổng) và hướng sửa cho lần sau:

| Loại lỗi | Frame | ID | Ghi chú |
| --- | --- | --- | --- |
| Mở track sớm (FP) | 51–78 | track 5 | nguồn chính của 80 FP; lần sau mở track muộn hơn |
| Giữ box muộn (FP) | 149–151, 169–171 | track 4, 8 | bỏ frame đuôi sau khi xe đã ra khung |
| Box lỏng khi bị che | 82–115 | track 5, 6 | siết bbox ôm sát phần thấy được |

## 4. Kết quả model: ByteTrack control vs ReID treatment

Cấu hình từ `outputs/model_run_config.json`:

| Mục | Giá trị |
| --- | --- |
| Python / ultralytics / torch / lap | 3.13.15 / 8.4.145 / 2.11.0+cu128 / 0.5.13 |
| weights / hai tracker | yolo26n.pt / `bytetrack.yaml` (control), `botsort-reid.yaml` (treatment) |
| conf / IoU / imgsz / classes | 0.25 / 0.7 / 960 / [2, 5, 7] (car, bus, truck) |
| device | 0 (GPU) |

Sản lượng model: ByteTrack 607 bbox · 16 track; BoT-SORT+ReID 638 bbox · 16 track (nhãn tay: 646 bbox · 8 track).

| So sánh | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| bạn vs gold | 0.768 | 0.743 | 0.796 | 0.869 | 0.929 | 0.848 | 0.856 | 80 | 7 | 0 |
| ByteTrack control vs gold | 0.709 | 0.649 | 0.776 | 0.846 | 0.875 | 0.749 | 0.823 | 88 | 54 | 2 |
| BoT-SORT + ReID vs gold | 0.764 | 0.711 | 0.820 | 0.872 | 0.900 | 0.792 | 0.860 | 91 | 26 | 2 |
| ReID vs bạn | 0.746 | 0.690 | 0.807 | 0.905 | 0.863 | 0.732 | 0.898 | 81 | 89 | 3 |

Cổng: nhãn tay **qua**, ByteTrack **trượt** (MOTA 0.749 < 0.75), ReID **qua**.

### Stretch — quét ngưỡng ReID (`appearance_thresh`)

| appearance_thresh | HOTA | DetA | AssA | IDF1 | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0.70 | 0.763 | 0.711 | 0.820 | 0.900 | 91 | 26 | 2 |
| 0.80 | 0.763 | 0.711 | 0.820 | 0.900 | 91 | 26 | 2 |
| 0.90 | 0.763 | 0.710 | 0.820 | 0.899 | 91 | 27 | 2 |

Nhận xét: nới ngưỡng appearance từ 0.70 lên 0.90 gần như không đổi kết quả — 0.70 và 0.80 trùng khít, 0.90 chỉ nhích FN 26→27 và IDF1 0.900→0.899. Trên clip này cue appearance không phải yếu tố quyết định: các đoạn cắt/che ít và ngắn nên motion + IoU đã xử lý gần hết; chỉnh riêng ngưỡng ReID không mua thêm được identity.

## 5. Phân tích — năm câu hỏi

**1. MOTA của bạn cao hơn hay thấp hơn IDF1? Nếu MOTA cao mà IDF1 thấp thì điều đó nói gì, và vì sao MOTA không phạt nặng lỗi ID?**

MOTA của tôi (0.848) **thấp hơn** IDF1 (0.929). MOTA = 1 − (FP + FN + IDSW)/GT = 1 − (80 + 7 + 0)/573 = 0.848; nó bị kéo xuống bởi 80 FP (do mở track sớm), tức là lỗi **đếm detection**, không phải lỗi ID. Vì IDSW = 0 nên IDF1 gần như trọn vẹn.

Trường hợp ngược lại — MOTA cao mà IDF1 thấp — nghĩa là số lượng box đúng (ít FP/FN) nhưng **danh tính sai** (nhiều ID switch / track bị chia). MOTA không phạt nặng lỗi ID vì mỗi lần đổi ID chỉ bị tính **một sự kiện** tại frame xảy ra; các frame sau đó object vẫn được coi là "matched" nên không bị trừ tiếp. IDF1 thì phạt toàn bộ quãng bị gán nhầm danh tính. Vì vậy một cú switch giữa track gần như không làm xê dịch MOTA nhưng có thể làm sụp IDF1 của track đó.

**2. ByteTrack control và BoT-SORT + ReID treatment khác nhau thế nào ở IDF1, AssA và IDSW? Dẫn một frame sequence để giải thích, và vì sao đây không phải causal effect cô lập của ReID.**

- IDF1: 0.875 → 0.900 (+0.025)
- AssA: 0.776 → 0.820 (+0.044)
- IDSW: 2 → 2 (không đổi)

Treatment cải thiện độ nhất quán danh tính (IDF1, AssA tăng) nhưng số cú switch tuyệt đối vẫn bằng nhau. Frame sequence: đoạn hai xe cắt/che nhau — track tham chiếu 5 & 6 quanh **frame 105–115**. ByteTrack chia nhỏ track 6 và chỉ phủ 42/56 frame (0.75); ReID phủ 44/56 (0.79) và nối lại đúng xe sau khi tách nhờ embedding ngoại hình → AssA cao hơn. IDSW vẫn là 2 vì cả hai vẫn xử lý sai đúng một lần cắt nhau.

Đây **không** phải causal effect cô lập của ReID: ByteTrack và BoT-SORT khác nhau ở cả motion model, matching cascade và bù chuyển động camera. Nên delta đo được trộn lẫn "BoT-SORT vs ByteTrack" với "bật/tắt ReID" — không thể quy toàn bộ cải thiện cho riêng ReID.

**3. DetA, FP và FN đổi thế nào? Lỗi còn lại là detector hay association?**

- DetA: 0.649 → 0.711 (+0.062)
- FP: 88 → 91 (gần như không đổi)
- FN: 54 → 26 (giảm hơn một nửa)

ReID chủ yếu **hồi phục recall** (bắt được nhiều xe bị bỏ sót hơn), FP gần như giữ nguyên. Lỗi còn lại là **cả hai nhưng thiên về association**: FN vẫn còn 26 (so với nhãn tay chỉ 7) → detector còn miss xe nhỏ/bị che; và cả hai model đều tách 8 track thật thành **16 pred track** — đó là lỗi fragmentation, tức association. Sau khi recall được cải thiện, khoảng cách lớn nhất còn lại so với nhãn người nằm ở association.

**4. Một chỗ bạn đúng và ReID sai (frame, ID, vì sao):**

**Frame 87, track 5.** ReID đổi ID track 5 (từ pred 17 sang 18) ngay sau một đoạn bị che ngắn. Nhãn của tôi giữ nguyên một ID xuyên suốt track 5 (IDSW = 0, không phân mảnh). Tôi đúng vì nhìn bằng mắt xác nhận vẫn là cùng một xe; ReID sai vì embedding sau che khớp nhầm sang ID mới.

**5. Một chỗ ReID làm bạn xem lại annotation (frame, ID, vì sao):**

**Frame 51–78, track 5.** Bản của tôi có bbox track 5 từ frame 51, nhưng cả gold lẫn hai model đều không đặt box sớm như vậy. Bảng xếp hạng frame bất đồng nhất giữa ReID và nhãn tay cũng chỉ đúng đoạn này — frame 51, 52, 54 tôi có 2 box, ReID chỉ 1. Bằng chứng đó cho thấy tôi **mở track quá sớm** khi xe còn nhỏ/mờ — nguồn phần lớn 80 FP. Nó khiến tôi xem lại và siết luật: chỉ mở track khi xe chắc chắn là 4 bánh và đã vào khung đủ.

## 6. Nếu phải gán thêm 10 clip nữa

Sửa trong `GUIDELINE_MINI.md`:
- Ghi rõ ngưỡng start/end bằng số: box đầu tiên chỉ đặt khi xe ≥ nửa thân trong khung và chắc chắn là 4 bánh; box cuối là frame cuối xe còn thấy thật, không có frame đuôi. (Đây là nguồn ~80 FP.)
- Thêm luật keyframe: đặt dày ở đoạn vào/ra khung và đoạn bị che, để bbox không lỏng lúc chồng lấn.

Đổi trong quy trình:
- Dùng output model (BoT-SORT + ReID) làm **pre-label** rồi sửa tay, thay vì vẽ từ đầu — tiết kiệm thời gian và giảm miss.
- Thêm một lượt review chuyên biệt cho **biên track** (3 frame đầu + 3 frame cuối của mọi track), vì lỗi của tôi tụ ở đó.
- Chạy báo cáo loose-box (IoU thấp) trước khi khóa pre-gold để bắt box lỏng sớm.

## 7. Tệp đã nộp

- [x] `annotations/clip_01/gt.txt`
- [x] `annotations/clip_02/gt.txt`
- [x] `evidence/pre-gold/clip_01/gt.txt` và `manifest.json`
- [x] `GUIDELINE_MINI.md` 
- [x] `outputs/eval_vs_gold.json`
- [x] `outputs/model_bytetrack_clip_01.txt`
- [x] `outputs/model_reid_clip_01.txt`
- [x] `outputs/model_run_config.json`
- [x] `outputs/eval_bytetrack_vs_gold.json`, `outputs/eval_reid_vs_gold.json`, `outputs/eval_reid_vs_me.json`
- [ ] `reports/review_partner.md` — không có (làm cá nhân)
- [x] `reports/REPORT.md` 
