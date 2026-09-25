# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Đức Hà

Công cụ gán nhãn đã dùng: CVAT

## 1. Dữ liệu và cách chia tập

Các ảnh được lấy từ cùng một video có camera cố định. Hai ảnh gần nhau có thể chứa cùng một xe và gần như giống nhau. Vì vậy, pool và test được chia theo thời gian, có vùng đệm ở giữa.

Nếu chia ngẫu nhiên, các ảnh gần giống nhau có thể nằm ở cả train và test. Khi đó điểm test sẽ cao hơn thực tế vì mô hình đã học cảnh gần giống ảnh kiểm tra. Chia theo thời gian giúp kết quả đánh giá khách quan hơn.

## 2. Mô hình khởi đầu lạnh

| vòng | model | ảnh train | box train | AP50 | chênh so cold start | P | R | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | - | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Mô hình có precision cao nhưng recall chỉ đạt 0.489, nghĩa là dự đoán khá chính xác nhưng còn bỏ sót nhiều xe. Xe nhỏ có recall thấp nhất, chỉ 0.182. Xe trung bình và xe lớn lần lượt đạt 0.547 và 0.561. Trong `compare_round0.jpg`, `frame_0050` có 11 TP, 2 FP và 7 FN.

Các xe ở xa chỉ thấy cụm đèn hậu là trường hợp cần xem lại trước khi kết luận mô hình sai. Nhãn test do một mô hình khác tạo và chưa được người rà từng box. Ánh đèn hoặc phản chiếu có thể bị nhầm là xe, còn các box cao dưới 16 pixel được bỏ qua khi chấm.

## 3. Chiến lược chọn mẫu

Công thức chọn mẫu là `score = 0.5*U + 0.3*A + 0.2*D`:

- `U` là độ bất định, cao khi confidence của box gần 0.5.
- `A` thể hiện số box mơ hồ cần kiểm tra trong ảnh.
- `D` là khoảng cách thời gian tới ảnh đã được gán nhãn gần nhất. Ở vòng đầu, chưa có ảnh đã gán nên `D = 1` cho tất cả ảnh.

`MIN_GAP_S = 2` giúp tránh chọn hai ảnh quá gần nhau. Ba ảnh đã chọn là `frame_0182` (hạng 1, điểm 0.9591, 28 box/18 box mơ hồ), `frame_0369` (hạng 2, điểm 0.9324, 43/16) và `frame_0187` (hạng 10, điểm 0.8995, 39/17). Ảnh có nhiều box mơ hồ sẽ tốn công rà hơn.

`frame_0372` đứng hạng 6 với điểm 0.9101 nhưng không được chọn. Ảnh này chỉ cách `frame_0369` 1,2 giây, nhỏ hơn khoảng cách 2 giây. Điểm cao chỉ cho biết ảnh nên được ưu tiên kiểm tra, không bảo đảm ảnh đó sẽ làm mô hình tốt hơn.

## 4. Các vòng học chủ động

| vòng | model | ảnh train | box train | AP50 | chênh so cold start | P | R | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | - | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 353 | 0.701 | -0.070 | 1.000 | 0.196 | 0.328 | 0.000 | 0.179 | 0.634 |

Vòng 0 chưa có dữ liệu train nên không có bước sửa nhãn. Ở vòng 1, model đề xuất 169 box trên 12 ảnh. Sau khi rà, bộ nhãn có 353 box: 89 box gần như được giữ nguyên, 59 box được sửa, 21 box bị xóa và 205 box được thêm.

Sau khi fine-tune, AP50 giảm từ 0.771 xuống 0.701. Recall xe nhỏ giảm từ 0.182 xuống 0, xe trung bình giảm từ 0.547 xuống 0.179, còn xe lớn tăng từ 0.561 lên 0.634. Như vậy model tốt hơn với xe lớn nhưng bỏ sót nhiều xe nhỏ và trung bình hơn.

Một ví dụ là `frame_0050`. Cold start có 11 TP, 2 FP, 7 FN; vòng 1 có 5 TP, 0 FP, 13 FN. Vòng 1 không còn FP nhưng mất thêm 6 TP. Kết quả này cho thấy model dự đoán ít xe hơn ở cùng ngưỡng 0.25. Chưa đủ bằng chứng để nói 205 box thêm mới là nguyên nhân.

Trong `BLIND_SCAN.md`, tôi đếm được 26 xe ở `frame_0187`. Phần mô tả chi tiết hai vị trí khó có tham khảo hỗ trợ nên không được xem là quan sát hoàn toàn độc lập. Khi xem pre-label, frame này có 14 box; nhãn cuối có 26 box, gồm 9 accepted, 4 edited, 1 deleted và 13 added. `REVIEW_LOG.csv` ghi một xe được thêm, một xe sát mép phải được sửa box và một xe lớn được giữ nguyên. Đây là việc sửa nhãn train; còn `compare_round1.jpg` là kết quả của model sau khi train.

Ca xe sát mép phải là một trường hợp khó. Theo guideline, tôi chỉ khoanh phần xe còn nhìn thấy và không kéo box ra ngoài ảnh. Với xe quá xa chỉ còn hai đèn và box cao dưới khoảng 16 pixel, có thể gán hoặc bỏ qua nhưng phải làm nhất quán.

## 5. Kết luận và giới hạn

Vòng 1 kém hơn cold start trên tập test vì AP50 giảm 0.070, recall và F1 cũng giảm. Tôi sẽ dừng train thêm để kiểm tra lại nhãn trước.

Nếu làm vòng sau, tôi sẽ ưu tiên hai nhóm:

1. Xe nhỏ ở xa và các cụm đèn hậu gần nhau. Nhóm này khó nhìn, tốn thời gian phân biệt xe với ánh phản chiếu.
2. Xe mờ hoặc bị cắt ở mép ảnh. Nhóm này cần kiểm tra kỹ phần thân còn nhìn thấy và độ chặt của box.

Với cả hai nhóm, tôi sẽ tránh chọn nhiều ảnh gần nhau vì chúng có thể chứa cùng một xe và không cung cấp nhiều thông tin mới.

Tập test chỉ có 20 ảnh. Trong 417 box tham chiếu có 14 box quá nhỏ bị bỏ qua, và toàn bộ nhãn tham chiếu chưa được người rà thủ công. Vì vậy, kết quả cho thấy vòng 1 khớp bộ nhãn test kém hơn, nhưng chưa đủ để khẳng định chất lượng ngoài thực tế cũng giảm đúng như vậy.

Trước khi train tiếp, tôi sẽ kiểm tra lại 12 ảnh train, nhất là 205 box thêm mới và 59 box đã sửa. Tôi sẽ kiểm tra box trùng, box lệch, xe nhỏ, xe sát mép và class id. Tôi không sửa `data/test/labels`; nếu thấy nhãn test đáng ngờ, tôi chỉ ghi lại frame và lý do.
