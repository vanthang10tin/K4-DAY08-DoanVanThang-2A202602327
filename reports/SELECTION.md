# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:
Nếu chỉ được ưu tiên 5 ảnh, tôi sẽ chọn:
1. `frame_0182.jpg` (hạng 1, t = 72.8s, score = 0.9591): điểm không chắc chắn U và số box lưỡng lự A đều cao nhất (U = 0.9182, A = 1.0, 28 box với 18 box không chắc).
2. `frame_0369.jpg` (hạng 2, t = 147.6s, score = 0.9324): mật độ xe cao (43 box), điểm phân vân cao (U = 0.9315, A = 0.8889).
3. `frame_0380.jpg` (hạng 3, t = 152.0s, score = 0.9170): cách xa frame 0369 hơn 4 giây, có 40 box và độ bất định cao (U = 0.9340).
4. `frame_0326.jpg` (hạng 4, t = 130.4s, score = 0.9155): điểm U = 0.9310, có 39 box với nhiều ca khó nhận diện ở xa.
5. `frame_0331.jpg` (hạng 5, t = 132.4s, score = 0.9154): cách frame 0326 đúng 2.0s, có tới 47 box và A = 1.0.
Cân nhắc ảnh gần trùng: Mặc dù `frame_0187.jpg` xếp hạng 10 (score = 0.8995), tôi không ưu tiên lấy trong top 5 vì nó chỉ cách `frame_0182.jpg` đúng 2.0s, bối cảnh giao thông ít biến đổi hơn so với việc dàn trải sang các cụm thời gian khác.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:
- `frame_0182.jpg` (hạng 1, score = 0.9591): CSV ghi nhận 18 box không chắc trên tổng 28 box, ảnh thể hiện nhiều xe ở làn đường xa và xe bị ngược sáng.
- `frame_0099.jpg` (hạng 8, score = 0.9063, t = 39.6s): CSV ghi nhận U = 0.9460 rất cao, ảnh có xe ở góc mép bị cắt và nhiều xe ở xa chỉ nhìn thấy đèn hậu.
- `frame_0107.jpg` (hạng 14, score = 0.8876, t = 42.8s): CSV ghi nhận 33 box với 15 box ambiguous, ảnh có nhiều xe chạy tốc độ cao và ánh đèn phản chiếu trên mặt đường.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:
- `frame_0372.jpg` (hạng 6, score = 0.9101, t = 148.8s): Có điểm cao hơn nhiều ảnh trong lô được chọn (như frame 0107 hay frame 0392), nhưng bị thuật toán loại bỏ vì chỉ cách `frame_0369.jpg` (t = 147.6s) đúng 1.2 giây (nhỏ hơn ngưỡng tối thiểu `min_gap_s = 2.0s`). Việc loại bỏ này là hợp lý để tránh tốn công sửa hai ảnh gần như trùng lặp góc nhìn.

Điều phép chọn này chưa chứng minh về chất lượng mô hình:
Điểm số chọn mẫu (score) cao chỉ phản ánh mức độ không chắc chắn (uncertainty) và sự phân vân của mô hình đối với các vùng ảnh hiện tại, kết hợp với tính đa dạng thời gian. Điểm cao không đồng nghĩa hay chứng minh được rằng sau khi con người sửa nhãn cho các ảnh này thì mô hình chắc chắn sẽ học tốt hơn hay tăng điểm mAP tổng thể trên tập kiểm thử (test set).
