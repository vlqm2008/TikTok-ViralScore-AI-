# HƯỚNG DẪN QUY TRÌNH XỬ LÝ DỮ LIỆU & HUẤN LUYỆN MÔ HÌNH DỰ ĐOÁN ĐIỂM VIRAL TIKTOK

## 1. TỔNG QUAN KIẾN TRÚC HỆ THỐNG

Hệ thống được thiết kế theo quy trình xử lý dữ liệu và huấn luyện tự động (End-to-End Pipeline) gồm 6 chặng chính:

+-------------------+      +------------------------+      +-----------------------+
|  Chặng 1: Thu thập | -->  | Chặng 2: Bóc tách     | -->  | Chặng 3: Sửa lỗi ID   |
|  & Đối soát      |      | Đặc trưng Đa phương thức|      | & Ghép nối Inner Join |
+-------------------+      +------------------------+      +-----------------------+
                                                                       |
+-------------------+      +------------------------+                  v
|  Chặng 6: Huấn   | <--  | Chặng 5: Chia dữ liệu  | <--  +-----------------------+
|  luyện 2 Tầng     |      | Creator-Disjoint       |      | Chặng 4: Lọc rác &    |
+-------------------+      +------------------------+      | Chuẩn hóa Gauss       |
                                                           +-----------------------+

---

## 2. CHI TIẾT QUY TRÌNH THỰC THI

### CHẶNG 1: THU THẬP & ĐỐI SOÁT DỮ LIỆU THÔ (DATA AUDITING)

- Kho Video MP4: Thu thập tổng cộng 7,915 video MP4 độc bản (đã lọc trùng lặp từ danh sách ban đầu hơn 8,000 video do có 644 video gán trùng 2 chủ đề cùng lúc).
- Kho Số liệu Tương tác (Metadata): Thu thập từ 2 file CSV gốc đặt tại D:\DataGoc:
  - training_data_goc.csv: 5,819 dòng
  - streaming_data_goc.csv: 1,227 dòng
  - Tổng cộng: 5,819 + 1,227 = 7,046 dòng số liệu tương tác (Like, View, Share).
- Xử lý Chênh lệch Dữ liệu:
  - Có 869 video đã tải file MP4 nhưng thiếu số liệu tương tác (7,915 - 7,046 = 869). Nguyên nhân do chủ kênh đã xóa video hoặc chuyển sang chế độ riêng tư (Private) trước khi bot tiến hành cào số liệu.
  - Toàn bộ 869 video này bị mất nhãn (ground truth) nên bắt buộc loại bỏ khỏi tập huấn luyện.

---

### CHẶNG 2: BÓC TÁCH ĐẶC TRƯNG ĐA PHƯƠNG THỨC (MULTIMODAL EXTRACTION)

Quy trình tự động hóa qua script batch_pipeline.py, sử dụng Gemini Flash API làm bộ não phân tích:

1. Tiền xử lý Âm thanh & Hình ảnh:
   - FFmpeg: Tách âm thanh Mono 16kHz (độ trễ 0.15s).
   - OpenCV: Trích xuất tối đa 8 khung hình đại diện (key-frames) cho mỗi video.
2. Phân tích Đa phương thức (Xoay vòng Key Pool API):
   - Xử lý Âm thanh: Bóc băng toàn bộ lời thoại tiếng Việt (Transcript), tự động phân định ranh giới giữa lời nói tự nhiên (Speech) và lời bài hát (Lyrics).
   - Xử lý Hình ảnh & OCR: Đọc văn bản trên màn hình (OCR giỏ hàng, link bio, mã giảm giá), phát hiện nội dung quảng cáo (isAd) và đánh giá cảm xúc (ad_sentiment).
   - Quản lý Bộ nhớ: Tự động xóa bộ nhớ đệm hình ảnh tạm thời ngay sau khi hoàn thành từng video để tối ưu dung lượng đĩa cứng.
3. Trích xuất Hook (5 giây đầu): Chạy đối chứng chéo độc lập giữa Gemini Flash và Seedance 2.0 Mini để thiết lập 18 cờ bằng chứng Hook (câu hỏi gây tò mò, hiệu ứng thị giác giật gân, nhịp độ chuyển cảnh...).
4. Tổng hợp Đặc trưng: Script export_training_dataset.py gom 7,916 file JSON kết quả thành 1 bảng tập trung final_training_features.csv (chứa 27 đặc trưng AI).

---

### CHẶNG 3: SỬA LỖI ĐỊNH DẠNG ID & GHÉP NỐI DỮ LIỆU (DATA RESTORATION)

Thực thi qua script process_data_goc.py:

1. Khôi phục Lỗi Số mũ Excel:
   - Lỗi làm tròn số nguyên lớn của Excel đã biến đổi ID 19 chữ số thành dạng số mũ (ví dụ: 7.28E+18), làm mất liên kết hơn 3,600 video.
   - Sử dụng biểu thức chính quy (Regex) quét trực tiếp vào đường dẫn video vid_url với mẫu: r'/video/(\d+)' để trích xuất lại chính xác 100% ID nguyên bản.
2. Ghép nối Chính xác (Inner Join):
   - Ghép bảng số liệu tương tác sạch (7,046 video) với bảng đặc trưng AI (7,915 video) theo khóa chính video_id.
   - Kết quả thu được tập dữ liệu khớp 100%: merged_7046_multimodal.csv (7,046 video có đầy đủ hình ảnh, kịch bản và số liệu tương tác).

---

### CHẶNG 4: TIỀN XỬ LÝ & CHUẨN HÓA GAUSS (PREPROCESSING & NORMALIZATION)

Thực thi qua script src/01_preprocess_and_label.py:

1. Lọc Nhiễu Mẫu số Nhỏ:
   - Loại bỏ 45 video có lượt xem dưới 100 views (views < 100).
   - Lý do: Lượt xem quá nhỏ gây ra tỷ lệ tương tác ảo (ví dụ: 1 like trên 2 views tạo ra tỷ lệ 50%).
   - Dữ liệu sạch còn lại: 7,046 - 45 = 7,001 video chuẩn mực.
2. Chuẩn hóa Phân phối Gauss (Blom's INT):
   - Phân phối điểm tương tác gốc bị lệch đuôi nặng (đa số điểm thấp, một số ít video triệu view bị vọt lên quá cao).
   - Sử dụng công thức Blom's Inverse Normal Transform (INT) đưa phân phối điểm số về dạng hình quả chuông chuẩn mực N(0, 1).
3. Gán Nhãn Bứt phá (is_breakout):
   - Gán nhãn is_breakout = 1 cho các video có điểm tương tác cao vượt trội so với trung bình của chính kênh đó (Z-score >= 1.5).
4. Xuất Tập Dữ liệu:
   - dataset_with_breakout_labels.csv: 7,001 dòng (đầy đủ nhãn).
   - features_content_only.csv: 18 cột đặc trưng nội dung thuần túy.
   - features_full_fusion.csv: 68 cột đặc trưng đa phương thức toàn diện.

---

### CHẶNG 5: PHÂN CHIA DỮ LIỆU CHỐNG "HỌC VẸT" (CREATOR-DISJOINT SPLIT)

Thực thi qua script src/02_split_creator_disjoint.py:

- Áp dụng thuật toán GroupShuffleSplit để chia 7,001 video thành 3 tập độc lập dựa trên ID của kênh tạo nội dung (author_id):
  - Tập Train (70%): 4,964 video (thuộc 250 kênh) — Phục vụ huấn luyện mô hình.
  - Tập Validation (15%): 996 video (thuộc 54 kênh) — Phục vụ tinh chỉnh siêu tham số.
  - Tập Test (15%): 1,041 video (thuộc 54 kênh hoàn toàn mới) — Phục vụ kiểm thử độc lập.
- Ràng buộc Tuyệt đối: Kênh xuất hiện ở tập Train không được xuất hiện ở tập Test. Ràng buộc này ép AI phải học chất lượng kịch bản và âm thanh thực sự, tránh việc mô hình ghi nhớ độ nổi tiếng của các kênh lớn (Channel Bias).

---

### CHẶNG 6: HUẤN LUYỆN 2 TẦNG & GIẢI MÃ SHAP (TWO-STAGE TRAINING & SHAP)

Thực thi qua script src/03_two_stage_train_shap.py và src/04_evaluate_and_benchmark.py:

Tầng 1: Content-Only Model ----> Phân tích tầm quan trọng (TreeSHAP)
   (18 đặc trưng nội dung)
            |
            v (Điểm nội dung dự đoán)
Tầng 2: Full-Fusion Model ------> Dự đoán điểm Viral cuối cùng
   (Điểm nội dung + Thống kê kênh)

1. Kiến trúc 2 Tầng & Phân tích TreeSHAP:
   - Tầng 1 (Content-Only Model): Mô hình chỉ học trên 18 đặc trưng nội dung (bịt mắt không cho nhìn chỉ số Follower). Phân tích trọng số bằng TreeSHAP thu được tỷ lệ đóng góp thực tế:
     - Kịch bản & Cảm xúc (Script & Sentiment): 62.52%
     - Giọng nói & Âm thanh (Audio & Voice): 20.49%
     - Thị giác & Hook 5s (Visual & 5s Hook): 17.00%
   - Tầng 2 (Full-Fusion Model): Tích hợp điểm nội dung (Content Score) từ Tầng 1 với chỉ số quy mô kênh (Follower, baseline lịch sử) để dự đoán điểm Viral tổng thể.
2. Đánh giá trên Tập Test:
   - Đánh giá hiệu năng xếp hạng trên 54 kênh mới chưa từng thấy thông qua các chỉ số: Spearman, Kendall, và NDCG@5 (kết quả xuất ra thư mục results/).

---

## 3. HƯỚNG DẪN THỰC THI NHANH (QUICK START)

Toàn bộ pipeline từ Chặng 1 đến Chặng 6 có thể chạy tự động trong vòng dưới 25 giây:

- Chạy bằng Python:
  python run_pipeline.py

- Chạy bằng Batch File (Windows):
  Thực thi file run_pipeline.bat hoặc click đúp vào file trên màn hình.