# Báo cáo Day 09 — Phạm Trần Anh, 2A202604057

## 1. Kết luận trước
Pipeline kết luận **Hold**: nhận 224 ảnh, loại 31 ảnh tại intake, xuất 193 ảnh dưới dạng **bản nháp**. Không còn lỗi bắt buộc; sau khi duyệt mô phỏng 80 ảnh, vẫn còn 34/193 ảnh chưa được duyệt có cảnh báo (17,62%), vượt ngưỡng Hold 10%. Nguồn: `reports/01_intake.txt`, `reports/08_summary.json` và manifest của bản xuất.

## 2. Lỗi và phân việc
Bảng lấy từ `reports/02_error_list_v1.csv`; số lượng là số issue, có thể nhiều issue trên cùng một ảnh.

| Mã lỗi | Số issue | Ai xử lý | Lý do |
|---|---:|---|---|
| CORRUPT_IMAGE / BAD_SIZE | 2 / 3 | Máy | Loại ảnh hỏng hoặc không đủ kích thước |
| DUP_IMAGE_EXACT / DUP_IMAGE_NEAR / LEAK_TEST | 8 / 7 / 11 | Máy | Loại ảnh trùng, rò rỉ test/val |
| MISSING_LABEL_FILE | 6 | Người | Máy không tạo nhãn thay người |
| BAD_FORMAT / BAD_CLASS_ID / BOX_ZERO / DUP_BOX | 3 / 6 / 4 / 7 | Máy | Xoá dòng lỗi, ghi log và chuyển kiểm tra tiếp |
| BOX_OUT_OF_IMAGE | 8 | Máy | Cắt box về biên ảnh |
| BAD_CLASS_NAME | 2 | Máy | Chuẩn hoá names theo cấu hình |
| EMPTY_LABEL / BOX_HUGE / BOX_TINY | 11 / 3 / 6 | Người | Cần kiểm tra ý nghĩa và tính hợp lệ |
| CLASS_RARE | 4 | Người | Kiểm tra phân bố lớp cấp dataset |
| POSSIBLE_MISSING / POSSIBLE_LOOSE_BOX | 53 / 27 | Người | Ý kiến model chỉ là cảnh báo |
| POSSIBLE_WRONG_CLASS / POSSIBLE_FALSE_POS | 17 / 28 | Người | Đối chiếu ảnh trước khi sửa |

`03_fixes.csv` ghi 35 thao tác: 20 drop_line, 8 clip_box, 1 rename_classes và 6 needs_label (chỉ chuyển người, không tạo file). Hàng đợi có 121 ảnh; 100 ảnh mang ít nhất một mã POSSIBLE_. Ảnh đầu `img_0020` có priority 11,524 do cộng trọng số nhiều dấu hiệu (FIXED_DROP_LINE, POSSIBLE_MISSING, POSSIBLE_WRONG_CLASS), cao hơn mức NEEDS_LABEL=10 của ảnh thiếu nhãn. Trong 80 ảnh duyệt mô phỏng, 74 ảnh có cảnh báo ngữ nghĩa; đây là số ảnh có cờ được xem, không phải số lỗi thực đã xác nhận. Nguồn: `05_review_queue.csv`, `06_review_log.csv`.

## 3. Calibration (2 ngưỡng)
Chạy `make calibrate`, chỉ sửa hai giá trị được phép trong cấu hình. P/R dưới đây lấy từ `calibration/calibration_report.md`.

| Tham số | Khởi đầu | Đã chọn | Precision / Recall trước → sau | Đánh đổi |
|---|---:|---:|---|---|
| duplicates.phash_dist | 2 | 16 | NEAR: 1,00/0,50 → 1,00/0,83 | Bắt 5/6 ảnh gần trùng, không có FP trên calib; giữ EXACT ở 1,00/1,00 |
| second_opinion.conf | 0,25 | 0,6 | WRONG_CLASS: 0,56/0,83 → 0,62/0,83; MISSING: 0,50/1,00 → 0,57/1,00 | Giảm báo động giả, giữ recall thiếu nhãn; LOOSE_BOX giảm 0,29/0,40 → 0,20/0,20 |

Calib chỉ có 39 ảnh nên precision 100% không bảo đảm không xoá nhầm trên raw. Sweep ở phash_dist=20 còn làm recall EXACT giảm còn 0,33: ngưỡng lớn có thể thay đổi ảnh được giữ và cách phân loại lỗi. Trên raw, ngưỡng 16 loại 7 ảnh gần trùng; `img_0011`–`img_0008` và `img_0084`–`img_0074` đều đúng sát biên khoảng cách 16. Các cặp này cần kiểm tra bằng mắt; báo cáo này chưa khẳng định đã xác nhận trực quan mọi cặp. Không dùng kết quả gate để ép chọn ngưỡng đạt Release.

## 4. Cửa chặn và điều pipeline không bắt được
Chạy đối chứng `python -m pipeline.run --budget 0 --reports work/rep_nb --export-dir work/exp_nb`; kết quả thực ghi trong `work/rep_nb/08_summary.json` và `work/gate_check.log`. Kết quả: **Rework**, mã thoát **20**, còn **6 lỗi bắt buộc** (thiếu file nhãn), 108/193 ảnh có cảnh báo (55,96%), **không xuất dataset**. Không sửa nhãn raw hoặc cấu hình cố định để thử gate.

Hai hạn chế có số liệu: (1) checker bỏ sót 4/5 lỗi box lỏng trên calib tại conf=0,6 (recall 20%); (2) phash vẫn bỏ sót 1/6 ảnh gần trùng trên calib, nên không thể kết luận đã loại hết ảnh trùng biến đổi mạnh. Ngoài ra FALSE_POS có 11 FP với 5 TP (precision 31%): model không thấy vật không có nghĩa nhãn người sai. Các ví dụ này chứng minh cần người duyệt và không thể coi 0 lỗi cú pháp là dữ liệu hoàn toàn sạch.

## 5. Train: raw vs clean (tùy chọn)
**Chưa chạy**: hiện hoàn thành pipeline và calibration, chưa thực hiện thí nghiệm Colab. Không có kết quả mAP hoặc độ lệch chuẩn giữa seed, nên chưa thể kết luận clean tốt hơn raw. Nếu train sau, dùng cùng cấu hình và hai seed cho hai biến thể, báo chênh lệch cùng độ nhiễu; chỉ dùng test để đánh giá cuối.

## 6. Quyết định và ghi chú AI
Giữ quyết định **Hold**, chỉ sử dụng bản xuất để thử nghiệm. Cần người xác minh các cảnh báo còn lại và các cặp ảnh gần trùng; việc phát hành phải đáp ứng quy tắc dự án. Không thay ngân sách 80 hay ngưỡng Hold 10% để làm đẹp kết quả.

- Prompt chính: yêu cầu AI hoàn thiện label_format, second_opinion, autofix, gate theo docstring và test; chọn đúng hai ngưỡng từ sweep và viết báo cáo bằng số liệu thật.

