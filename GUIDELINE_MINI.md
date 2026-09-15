# Mini annotation guideline — Ngày 3 (tracking)

Người gán nhãn: **Phong** (làm cá nhân)
Clip: **clip_01** (190 frame, 12.5 fps ≈ 15.2 giây)

> Tài liệu này ghi lại luật gán nhãn và các ca mơ hồ đã xử lý cho clip_01, để
> người đọc sau gán lại giống hệt.

---

## 1. Phạm vi: gán cái gì, không gán cái gì

Một lớp duy nhất: **`vehicle`** — xe bốn bánh (xe con, van, xe buýt, xe tải).

| Gán | Không gán |
| --- | --- |
| xe con, SUV, taxi, xe bán tải | người đi bộ |
| van, minivan | xe đạp |
| xe buýt, minibus | **xe máy / mô tô** |
| xe tải, xe đầu kéo | xe trong ảnh quảng cáo, trong gương, dưới bóng nước |

Ghi chú:
- Model chạy với `classes = [2, 5, 7]` (COCO: car / bus / truck) ở `conf = 0.25`, `imgsz = 960` — đúng bằng phạm vi trên. Xe máy (class 3), xe đạp (class 1), người (class 0) bị loại tự động.
- Xe máy xuất hiện trong nhãn tay là lỗi người gán, không phải lỗi model.

## 2. Luật ID — phần quan trọng nhất

| Tình huống | Luật | Vì sao |
| --- | --- | --- |
| Xe bị che một phần rồi hiện lại | giữ nguyên ID nếu bị che **dưới 25 frame** (25 frame = 2 giây @ 12.5 fps) | cùng một xe vật lý; re-ID cho đoạn che ngắn sẽ tạo ID switch giả. |
| Xe bị che lâu hơn 25 frame | giữ ID **nếu** quỹ đạo, hướng đi và kích thước nối tiếp rõ ràng; nếu mất dấu không chắc chắn → **track mới** | tránh vừa giữ nhầm ID (ID switch) vừa cắt nhầm track (fragment). |
| Xe rời khung hình rồi quay lại | **track mới** | không có gì đảm bảo là cùng xe; ID mới an toàn hơn ID đoán. |
| Hai xe cắt nhau / chồng lên nhau | giữ nguyên **cả hai** ID xuyên suốt lúc chồng; sau khi tách, gán lại theo **quỹ đạo trước–sau**, không theo vị trí | ID bám xe vật lý, không bám ô vị trí. |

## 3. Luật bbox

| Tình huống | Luật |
| --- | --- |
| Xe bị cắt bởi rìa ảnh | bbox chạm đúng rìa, không đoán phần ngoài ảnh |
| Xe bị xe khác che một phần | bbox ôm **phần nhìn thấy được**, không ước lượng cả thân xe |
| Xe vừa xuất hiện, còn nhỏ / mờ | bắt đầu track ở frame đầu tiên xe **chắc chắn là 4 bánh và đã vào khung ~≥ nửa thân** — không mở track sớm hơn |
| Xe đang đỗ, không di chuyển | vẫn gán **mọi frame**, giữ ID cố định; lớp là `vehicle` bất kể đứng hay chạy |
| Keyframe đặt dày ở đâu | dày ở lúc **vào/ra khung**, lúc **bị che / chồng nhau**, lúc **đổi hướng**; thưa ở đoạn đi thẳng đều |

## 4. Các ca mơ hồ đã gặp và cách xử lý

### Ca 1 — bắt đầu track khi xe mới vào khung
- clip_01 / frame 51–78 / track 5 (tương tự: track 4 f50–53, track 6 f78–100, track 7 f102–105, track 8 f133–135)
- Tình huống: xe mới vào khung, còn nhỏ / chưa rõ hẳn — chưa chắc đặt bbox đầu tiên ở frame nào.
- Quyết định: **mở track muộn hơn**, chỉ khi xe rõ là 4 bánh và đã vào khung đủ.
- Lý do: mở sớm là nguồn chính của false positive.

### Ca 2 — hai xe chồng lấn / bị che
- clip_01 / frame 82–115 / track 5 & track 6
- Tình huống: hai xe cắt/che nhau; box dễ vẽ rộng ôm cả xe.
- Quyết định: giữ nguyên ID cả hai; **siết box ôm sát phần pixel thấy được**.
- Lý do: giữ IoU với gold cao, tránh box lỏng khi bị che.

### Ca 3 — xe rời khung / cuối clip
- clip_01 / frame 149–151 (track 4), 169–171 (track 8), 190 (track 1)
- Tình huống: xe ra mép khung hoặc clip kết thúc.
- Quyết định: dừng track ở **frame cuối cùng xe còn thấy thật**, không giữ box thêm 2–3 frame đuôi.
- Lý do: mỗi frame đuôi thừa là một false positive.

## 5. Đối chiếu với gold và điều chỉnh

Điểm chấm nhãn tay vs gold clip_01: IDF1 **0.929**, MOTA **0.848**, MOTP **0.856**, HOTA **0.768**, IDSW **0**, FP **80**, FN **7** — cả 3 cổng PASS.

Nhận xét chính: có **646 box vs gold 573 (+73)**, gần như toàn bộ chênh do mở track sớm và giữ box muộn. Ngược lại, kỷ luật ID rất tốt: IDSW = 0, không phân mảnh, đúng 8 track = gold.

Điều chỉnh áp dụng cho lần gán sau:
- Mở track muộn hơn (sửa Ca 1) — cắt phần lớn 80 FP.
- Kết thúc track đúng frame cuối xe còn thấy, bỏ frame đuôi (Ca 3).
- Siết bbox lúc bị che ở track 5/6, f82–115 (Ca 2).
- Giữ nguyên cách gán ID hiện tại.

Bối cảnh: nhãn tay (MOTA 0.848) cao hơn cả ByteTrack (0.749, trượt cổng MOTA) và BoT-SORT+ReID (0.792) khi cùng chấm trên gold — nhãn người là chuẩn để đo model.
