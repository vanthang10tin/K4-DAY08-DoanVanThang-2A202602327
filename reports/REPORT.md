# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Đoàn Văn Thắng (MSSV: 2A202602327)

Công cụ gán nhãn đã dùng: CVAT Docker trên máy cá nhân

## 1. Dữ liệu và cách chia tập

Dữ liệu của bài toán được thu thập từ camera giám sát giao thông lắp cố định trên đường cao tốc vào ban đêm. Vì camera đứng yên tại một vị trí, một phương tiện di chuyển qua khung hình sẽ xuất hiện liên tục trong nhiều frame liên tiếp trong khoảng thời gian từ vài giây đến hàng chục giây.

Tập chưa gán nhãn (pool set) và tập kiểm thử (test set) bắt buộc phải được chia theo trục thời gian (temporal split) và có một vùng đệm an toàn (buffer zone) ở giữa, thay vì chia ngẫu nhiên (random split). Lý do là vì:
- Nếu chia ngẫu nhiên, các frame liền kề nhau của cùng một chiếc xe sẽ bị phân tán vào cả tập học (train/pool) và tập kiểm thử (test). Hiện tượng này gây ra rò rỉ dữ liệu (data leakage) nghiêm trọng: mô hình sẽ gặp lại chính chiếc xe đó với cùng góc quay, điều kiện ánh sáng và vận tốc mà nó vừa học.
- Khi bị rò rỉ dữ liệu, các số đo đánh giá trên tập kiểm thử (như mAP/AP50, Precision, Recall) sẽ bị lệch lạc quan (overly optimistic / bị thổi phồng nhân tạo). Điểm số trên giấy tờ sẽ rất cao nhưng hoàn toàn là điểm số ảo, không phản ánh đúng năng lực tổng quát hóa (generalization) của mô hình khi triển khai thực tế trên các luồng giao thông mới.

Việc chia tách theo trục thời gian kèm vùng đệm bảo đảm tập test hoàn toàn độc lập về mặt thời gian và danh tính xe so với tập huấn luyện.

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng số liệu vòng 0 từ `reports/rounds_table.md`:
```text
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
```

Dựa vào ảnh đối chiếu `outputs/compare_round0.jpg` và số đo trong `outputs/metrics_round0.json`:
- Mô hình khởi đầu lạnh (YOLOv8n nguyên bản huấn luyện trên tập COCO) không khớp tốt với nhãn tham chiếu ở các nhóm xe: xe ở làn đường xa chỉ nhìn thấy hai chấm đèn hậu nhỏ, xe bị che khuất một phần bởi dải phân cách hoặc mép ảnh, và các xe bị ánh đèn pha chói lóa ngược chiều. Ngoài ra, mô hình còn bị dương tính giả (false positive) ở một số vệt sáng phản chiếu đèn xe trên mặt đường ướt.
- Độ phủ (Recall tại conf 0.25) theo kích thước xe thể hiện rõ xu hướng:
  + Xe nhỏ (R small): đạt `0.1818` (18.18%, tức chỉ phát hiện được 12 trên 403 box tham chiếu).
  + Xe vừa (R medium): đạt `0.5473` (54.73%).
  + Xe lớn (R large): đạt `0.5610` (56.10%).
  Số liệu chứng minh độ phủ tỉ lệ thuận với kích thước vùng ảnh: xe càng nhỏ ở xa thì mô hình càng bỏ sót nghiêm trọng do thiếu đặc trưng chi tiết trong đêm.
- Một trường hợp cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai: Ở các vị trí rất xa gần chân trời nơi chỉ có hai chấm sáng mờ, nhãn tham chiếu (reference labels) cũng được sinh tự động từ mô hình máy học và chưa có chuyên viên kiểm định thủ công 100%. Đôi khi nhãn tham chiếu khoanh nhầm đèn đường/biển báo hoặc bỏ sót xe thật, do đó cần người rà soát trực tiếp hình ảnh gốc trước khi khẳng định mô hình dự đoán sai.

## 3. Chiến lược chọn mẫu

Công thức tính điểm ưu tiên chọn mẫu:
$$\text{Score} = W_U \cdot U + W_A \cdot A + W_D \cdot D$$
Trong đó:
- $U$ (Uncertainty): Mức chưa chắc chắn của mô hình, lấy trung bình từ tối đa 5 khung có mức chưa chắc chắn cao nhất trong ảnh.
- $A$ (Ambiguity): Số khung có độ tin cậy từ 0,15 đến dưới 0,50, chia cho số khung như vậy lớn nhất trong tập ảnh đang xét.
- $D$ (Diversity): Khoảng cách thời gian đến ảnh đã gán nhãn gần nhất, tính tối đa 10 giây rồi chia cho 10. Ở vòng chọn đầu tiên, chưa có ảnh nào được gán nhãn nên mọi ảnh đều có $D = 1$.
- $W_U, W_A, W_D$: Mức đóng góp của từng thành phần vào điểm chung; mặc định lần lượt là $0.5$; $0.3$; $0.2$.
- `EMPTY_BONUS`: Những ảnh mà mô hình không dự đoán được khung nào còn được cộng thêm điểm thưởng này để đưa vào diện xem xét (tránh bỏ sót trường hợp mô hình hoàn toàn mù tịt).
- Vai trò của `MIN_GAP_S` (mặc định 2.0 giây): Là khoảng cách thời gian tối thiểu giữa hai ảnh được chọn vào lô. Vì camera đứng yên, các frame cách nhau dưới 2 giây có góc nhìn và xe lưu thông gần như trùng hệt nhau. `MIN_GAP_S` ngăn chặn việc lãng phí công sức của người gán nhãn vào các frame gần như trùng lặp (redundant frames). Nếu lọc xong mà chưa đủ số lượng ảnh cần chọn, công cụ sẽ nới khoảng cách này. Do đó, các ảnh được chọn vào lô không nhất thiết phải là 12 ảnh đứng đầu tuyệt đối theo điểm số.

Minh chứng chọn mẫu từ `reports/SELECTION.md` và `outputs/selection_round1.csv`:
- Ba frame thuộc lô 12 ảnh được chọn:
  + `frame_0182.jpg` (hạng 1, score = 0.9591): Có $U = 0.9182$ và $A = 1.0$, chứa tới 18 box phân vân trên tổng số 28 box.
  + `frame_0099.jpg` (hạng 8, score = 0.9063): Có độ bất định cực cao $U = 0.9460$, phản ánh nhiều xe bị cắt mép và xe ở xa mờ ảo.
  + `frame_0107.jpg` (hạng 14, score = 0.8876): Chứa 33 box xe chạy tốc độ cao, có vệt sáng đèn pha trên mặt đường cần người phân định.
- Một frame điểm cao nhưng bị loại: `frame_0372.jpg` (hạng 6, score = 0.9101, cao hơn cả frame 0099 và frame 0107). Frame này bị loại bỏ vì thời điểm $t = 148.8s$, chỉ cách `frame_0369.jpg` ($t = 147.6s$) đúng 1.2 giây (nhỏ hơn ngưỡng `MIN_GAP_S = 2.0s`). Việc loại bỏ này chứng minh hệ thống đã cân đối tốt giữa độ bất định và chi phí trùng lặp cảnh.

Điểm bất định cao KHÔNG chứng minh việc gắn nhãn ảnh đó chắc chắn sẽ cải thiện chất lượng mô hình:
Độ bất định cao chỉ cho biết mô hình hiện tại đang gặp khó khăn trước ảnh đó. Nếu ảnh có quá nhiều nhiễu thị giác (cháy sáng đèn pha, nhòe chuyển động quá nặng, góc khuất bất khả kháng), mô hình có thể không học được đặc trưng hữu ích mà ngược lại có thể bị phạt loss lớn, dẫn đến hiện tượng overfit hoặc trở nên quá thận trọng.

## 4. Các vòng học chủ động (active learning)

Bảng tổng hợp kết quả các vòng từ `reports/rounds_table.md`:
```text
| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 309 | 0.443 | -0.328 | 1.000 | 0.154 | 0.267 | 0.000 | 0.145 | 0.463 |
```

Phân tích chi tiết vòng 1:
- **Mức độ sửa nhãn gợi ý** (trích xuất từ `outputs/round1_diff.md` và `outputs/round1_diff.json`):
  Trong 12 ảnh gán nhãn, mô hình ban đầu đề xuất 169 box. Sau khi người rà soát trên CVAT, tổng số box đạt 309 box:
  + `accepted`: 150 box (tỷ lệ chấp nhận 89% số box đề xuất).
  + `edited`: 6 box (kéo chỉnh lại mép viền ôm sát thân xe).
  + `deleted`: 13 box (loại bỏ các false positive như vệt phản quang mặt đường, đèn đường, khung trùng).
  + `added`: 153 box (bổ sung các false negative nghiêm trọng: các xe ở làn xa và xe bị mép cắt).
- **Biến động AP50**:
  AP50 giảm từ 0.771 xuống 0.443 ($\Delta = -0.328$).
- **Nhóm xe tốt lên / xấu đi**:
  + Tốt lên: Độ chính xác Precision@0.25 đạt mức hoàn hảo `1.000` (100%), tăng từ `0.9249`, với số lượng dương tính giả FP giảm triệt để từ 16 xuống 0. Mô hình không còn đoán bừa hay nhận nhầm ánh sáng mặt đường thành xe.
  + Xấu đi: Độ phủ Recall@0.25 giảm từ `0.4888` xuống `0.1538`. Cụ thể: R small giảm từ 0.1818 về 0.0000; R medium giảm từ 0.5473 về 0.1453; R large giảm từ 0.5610 về 0.4634. Mô hình trở nên cực kỳ thận trọng và chỉ dự đoán khi độ tự tin đạt mức rất cao.
- **Đối chiếu ảnh kết quả `compare_round0.jpg` và `compare_round1.jpg`**:
  Ở ảnh kiểm thử, mô hình vòng 1 triệt tiêu hoàn toàn các box rác ở dải phân cách và mặt đường (FP = 0), nhưng đồng thời mất đi khả năng nhận diện các đốm xe nhỏ ở hậu cảnh xa. Lý do là vì việc huấn luyện trên tập nhỏ (12 ảnh, 309 box) trong 50 epoch với bộ nhãn chuẩn khắt khe đã làm mô hình phạt nặng các box không chắc chắn, dẫn đến xu hướng under-prediction khi ngưỡng tin cậy đánh giá cố định ở 0.25.
- **Phân biệt ba nguồn thông tin**:
  1. *Quan sát độc lập (`BLIND_SCAN.md` trên `frame_0099.jpg`)*: Mắt người đếm được 21 xe, dự báo trước hai vùng rủi ro là xe nhỏ chỉ thấy đèn hậu ở xa và xe gần bị cắt mép/nhòe tốc độ cao.
  2. *Lỗi pre-label đã sửa (`REVIEW_LOG.csv` và `round1_diff.md`)*: Thực tế pre-label chỉ có 13 box; người gán nhãn đã thêm 9 box xe bị bỏ sót và xóa 1 box vệt sáng, đưa số box chính xác về 21 box khớp hoàn toàn với quan sát độc lập.
  3. *Kết quả mô hình sau train*: Mô hình đã tiếp thu quy tắc loại trừ vật thể không phải xe (Precision = 1.0), nhưng cần được bổ sung thêm nhiều mẫu xe xa ở các vòng tiếp theo để kéo lại Recall.
- **Mô tả ca khó theo guideline**: Xe bị cắt mép khung hình chỉ nhìn thấy một phần đuôi và đèn hậu. Theo `GUIDELINE_LABEL.md`, chỉ khoanh đúng phần nhìn thấy được, không cố đoán phần xe ngoài ảnh.

## 5. Kết luận và giới hạn

- **Đánh giá kết quả vòng 1**: So với khởi đầu lạnh, vòng 1 có bước tiến lớn về mặt độ chuẩn xác (Precision 100%, không còn FP), nhưng suy giảm đáng kể về độ phủ đối với xe nhỏ và vừa (AP50 giảm còn 0.443).
- **Quyết định: TIẾP TỤC làm vòng 2**:
  Việc dừng lại ở vòng 1 là chưa hợp lý vì mô hình đang trong giai đoạn chuyển đổi thích nghi với bộ nhãn mới, cần thêm dữ liệu đa dạng để khôi phục Recall. Cần gán nhãn lô 12 ảnh tiếp theo trong `to_label/round2/` để giúp mô hình học thêm về các hình thái xe ở cự ly xa.
- **Đề xuất hai ca còn yếu cho vòng sau**:
  1. *Xe đi ngược chiều ở làn xa có đèn pha chói lòa*: Cần nhiều ví dụ để AI phân biệt được nguồn sáng đèn pha và hình khối cabin xe.
  2. *Xe bị che khuất một phần bởi xe tải phía trước*: Giúp mô hình nhận biết xe trong điều kiện chồng lấn ranh giới.
  - *Chi phí và nguy cơ*: Chi phí rà soát 12 ảnh mới tốn khoảng 35-45 phút. Cần tiếp tục duy trì lọc `MIN_GAP_S` để tránh lãng phí thời gian vào các frame trùng góc nhìn.
- **Giới hạn thực nghiệm**:
  + Tập kiểm thử chỉ có 20 ảnh với 403 box tham chiếu; kích thước tập test nhỏ khiến các chỉ số mAP/AP50 rất nhạy cảm với biến động cục bộ.
  + Luật đánh giá bỏ qua 14 box xe nhỏ dưới 16px và nhãn tham chiếu do mô hình tự sinh chưa qua kiểm duyệt thủ công 100%, dẫn tới khả năng một số dự đoán đúng của mô hình bị tính là sai hoặc ngược lại.
- **Biện pháp kiểm tra khi AP50 giảm**:
  Trước khi tiếp tục train vòng kế tiếp, tôi sẽ:
  1. Kiểm tra lại toàn bộ file `.txt` trong `labels/round1/` xem có tọa độ nào bị lỗi chuẩn hóa hay nhầm class không.
  2. Rà soát file `outputs/round1_diff.json` để kiểm tra phân bố kích thước box được thêm mới.
  3. Đánh giá lại ngưỡng confidence threshold (`conf_thr`) khi test để xem mô hình có thực sự bị mất khả năng phát hiện hay chỉ đơn giản là tự tin ở ngưỡng thấp hơn (ví dụ 0.15 - 0.20 thay vì 0.25).
