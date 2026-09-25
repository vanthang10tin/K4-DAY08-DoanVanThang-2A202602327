# Hướng dẫn làm bài Day 8, từ đầu đến lúc nộp

Bài này dùng ảnh đường cao tốc ban đêm. AI đã khoanh xe nhưng còn sót và khoanh sai. Bạn xem 12 ảnh, sửa khung, cho AI học lại, rồi viết báo cáo.

Làm trong thư mục dự án trên máy bạn.

Mỗi bước dưới đây đi theo thứ tự. Làm xong bước trước rồi mới sang bước sau.

## Các file trong thư mục reports

Bài nộp cần 6 file trong `reports`. Ba file đã có. Ba file bạn còn phải tạo.

| File | Tình trạng | Bạn làm gì |
| --- | --- | --- |
| `BLIND_SCAN.md` | Đã điền và đã khóa | Không sửa |
| `blind_lock.json` | Đã tạo khi khóa | Không sửa |
| `rounds_table.md` | Đã có dòng vòng 0 | Không viết tay. Colab ghi thêm dòng vòng 1 ở bước 13 |
| `REVIEW_LOG.csv` | Chưa có | Bước 12. Ghi ít nhất 3 ca sửa khung |
| `SELECTION.md` | Chưa có | Bước 14. Giải thích vì sao chọn ảnh |
| `REPORT.md` | Chưa có | Bước 15. Báo cáo 5 mục |

File có chữ `TEMPLATE` chỉ là mẫu. Copy ra file mới rồi điền. Không nộp file mẫu.

## Bước 1. Tạo chỗ nộp bài trên GitHub

1. Đăng nhập GitHub.
2. Mở repo mẫu của lớp, bấm **Use this template** → **Create a new repository**.
3. Đặt tên repo, chọn **Public**.
4. Cài GitHub Desktop. **File → Clone repository**, chọn repo vừa tạo, tải về máy.

Sau khi tải về, mọi bước sau làm trong thư mục dự án đó.

## Bước 2. Đóng gói dữ liệu để đưa lên Colab

1. Mở PowerShell trong thư mục dự án.
2. Gõ:

```powershell
python tools/make_data_zip.py
```

3. Trong thư mục dự án xuất hiện file `day8_data.zip`. Giữ file này trên máy. Chưa đưa file zip lên GitHub.

## Bước 3. Chạy AI lần đầu trên Google Colab

1. Vào https://colab.research.google.com/ và đăng nhập Google.
2. **File → Upload notebook**. Chọn file `notebooks\day8_active_learning.ipynb` trong thư mục dự án.
3. **Runtime → Change runtime type → GPU**.
4. Trong ô đầu tiên của notebook, sửa tên thành tên bạn. Giữ nguyên hai dòng kia:

```python
STUDENT_NAME = "Nguyen Van A"
AL_K = 12
STRATEGY = "uncertainty"
```

5. **Runtime → Run all**. Khi được hỏi, tải file `day8_data.zip` lên.
6. Chờ chạy xong. Colab cho tải file `day8_round0_out.zip` về máy.

## Bước 4. Giải nén kết quả lần chạy đầu

1. Để file `day8_round0_out.zip` trong thư mục dự án.
2. Giải nén ngay vào thư mục đó, cùng chỗ với thư mục `data` và `reports`. Nếu Windows hỏi ghi đè, chọn gộp thư mục.

Xong bước này thì có:

- `outputs\metrics_round0.json`
- `outputs\compare_round0.jpg`
- `outputs\selection_round1.csv`
- `outputs\selection_round1.jpg`
- 12 ảnh trong `to_label\round1\images\train`

12 ảnh đó là: `frame_0099`, `frame_0107`, `frame_0182`, `frame_0187`, `frame_0227`, `frame_0270`, `frame_0312`, `frame_0326`, `frame_0331`, `frame_0369`, `frame_0380`, `frame_0392`.

## Bước 5. Nhìn một ảnh trước khi xem khung của AI

1. Mở một file `.jpg` trong `to_label\round1\images\train`. Chỉ mở ảnh. Chưa mở file `.txt` hay `.json`.
2. Mở `reports\BLIND_SCAN.md`. Ghi tên ảnh, số xe bạn đếm được, và hai chỗ dễ sót hoặc dễ vẽ sai.
3. Trong PowerShell, ở thư mục dự án, gõ:

```powershell
python tools/lock_blind.py
```

Lệnh này khóa file. Sau đó không sửa `reports\BLIND_SCAN.md` và không sửa `reports\blind_lock.json`.

Bản hiện tại đã khóa với ảnh `frame_0099.jpg`. Giữ nguyên. File này đọc như sau:

- Dòng `Frame`: tên ảnh đã xem là `frame_0099.jpg`.
- Dòng `Số xe`: nhìn bằng mắt, đếm được khoảng 18 xe.
- Đoạn tiếp theo là hai chỗ khó. Chỗ 1: góc dưới bên phải có một xe bị mép ảnh cắt mất, chỉ còn đuôi và đèn hậu. Chỗ 2: gần chân cầu có vài xe rất xa, chỉ còn hai chấm đèn. Vệt sáng trên mặt đường bên trái là ánh đèn, không phải xe.

Bạn không viết thêm gì vào file này.

## Bước 6. Quy tắc khoanh xe

Chỉ khoanh xe từ 4 bánh trở lên: ô tô, SUV, bán tải, van, xe tải, xe buýt.

- Khung ôm sát thân xe, kể cả gương và đèn.
- Vệt sáng đèn pha trên mặt đường, biển báo, đèn đường: không khoanh.
- Xe bị che hoặc bị cắt mép ảnh: chỉ khoanh phần nhìn thấy.
- Hai xe sát nhau: hai khung riêng.
- Xe rất xa, chỉ còn hai chấm đèn: khoanh hay bỏ đều được.
- Xe bị nhoè vì đang chạy: vẫn khoanh phần thân bị nhoè.

Với mỗi ảnh, xem theo thứ tự: xe nào thiếu khung, khung nào không phải xe, khung nào trùng, khung nào bị lệch.

## Bước 7. Tách 12 ảnh để tải lên CVAT

Thư mục `to_label\round1\images\train` có cả ảnh `.jpg` và file `.json`. CVAT chỉ cần ảnh.

12 ảnh đã được copy sang:

`to_label\round1\cvat_images`

## Bước 8. Tạo task trên CVAT

1. Mở CVAT trên trình duyệt và đăng nhập.
2. Bấm **+** để tạo task mới. Đặt tên `day8-round1`.
3. Thêm một nhãn duy nhất, tên đúng chữ `car`. Kiểu khung là hình chữ nhật.
4. Chọn cả 12 ảnh trong thư mục `cvat_images` ở bước 7.
5. Tạo task, rồi mở job bên trong.

Nếu giảng viên đã tạo sẵn task, mở task đó và kiểm tra đủ 12 tên ảnh ở bước 4.

## Bước 9. Đưa khung AI có sẵn vào CVAT

1. Trong thư mục dự án, mở thư mục `to_label\round1`.

2. Chọn đúng bốn mục nằm trong thư mục đó:
   - file `data.yaml`
   - file `train.txt`
   - thư mục `images`
   - thư mục `labels`
3. Chuột phải → **Compress to ZIP file**. Đặt tên `round1.zip`.
4. Trong job CVAT, bấm menu **⋮** → **Upload annotations**.
5. Định dạng chọn **Ultralytics YOLO Detection 1.0**. Chọn `round1.zip` rồi upload.

Ảnh sẽ hiện các hình chữ nhật sẵn. Đó là khung AI đề xuất.

## Bước 10. Sửa khung trên 12 ảnh

CVAT có thanh ảnh ở dưới, từ ảnh 1 đến ảnh 12.

- Xe chưa có khung: chọn công cụ hình chữ nhật, vẽ khung ôm sát thân xe.
- Khung sai hoặc hai khung chồng lên cùng một xe: bấm khung đó, bấm Delete.
- Khung lệch: bấm khung, kéo các cạnh cho sát thân xe.

Sửa xong một ảnh thì bấm **Save**, rồi sang ảnh kế tiếp. Làm đến hết ảnh thứ 12.

## Bước 11. Tải kết quả từ CVAT về máy

1. Quay ra trang task. Bấm **Actions** → **Export task dataset**.
2. Định dạng vẫn là **Ultralytics YOLO Detection 1.0**.
3. Tải zip về và giải nén.
4. Tìm thư mục `labels\train` bên trong. Đó là 12 file `.txt` bạn vừa sửa.

Trong PowerShell, ở thư mục dự án, gõ lệnh sau. Đổi đường dẫn thành chỗ bạn vừa giải nén:

```powershell
python tools/pack_labels.py to_label/round1 --yolo-dir "C:\đường\dẫn\tới\labels\train"
```

Lệnh chạy xong, không báo lỗi, thì trong dự án có thư mục `labels\round1`.

## Bước 12. Viết `reports\REVIEW_LOG.csv`

Đây là sổ tay 3 việc bạn vừa làm trên CVAT. Mỗi việc một dòng. Không cần viết đoạn văn.

1. Copy `reports\REVIEW_LOG_TEMPLATE.csv` thành `reports\REVIEW_LOG.csv`.
2. Xóa dòng ví dụ `frame_0000.jpg`.
3. Ghi 3 dòng. Mỗi dòng có 5 ô, cách nhau bằng dấu phẩy:

- Ô 1 luôn là `1`.
- Ô 2 là tên ảnh, ví dụ `frame_0099.jpg`.
- Ô 3 là bạn đang nói về xe nào, viết ngắn.
- Ô 4 là một từ: `added` nếu bạn vẽ thêm khung, `deleted` nếu bạn xóa khung, `edited` nếu bạn kéo khung cho sát, `accepted` nếu khung AI đúng và bạn giữ nguyên.
- Ô 5 là một câu vì sao.

```text
round,frame_id,object,action,rule_or_reason
1,frame_0099.jpg,xe bị cắt ở góc dưới phải,added,Chỉ khoanh phần đuôi còn nhìn thấy trong ảnh
1,frame_0107.jpg,vệt đèn trên mặt đường,deleted,Không phải xe nên xóa khung
1,frame_0182.jpg,xe tối sát mép trái,edited,Kéo khung ôm sát thân xe, kể cả đèn
```

Ba dòng trên chỉ là mẫu. Đổi thành đúng ảnh và đúng việc bạn đã bấm trên CVAT.

## Bước 13. Cho AI học lại trên Colab

1. Trong PowerShell, ở thư mục dự án:

```powershell
python tools/make_data_zip.py
```

File `day8_data.zip` được làm mới. Dùng file mới này.

2. Mở lại notebook trên Colab. **Runtime → Run all**. Tải `day8_data.zip` mới lên.
3. Tải về `day8_round1_out.zip`.
4. Giải nén vào thư mục dự án. Giữ file vòng 0: `metrics_round0.json` và `compare_round0.jpg`.
5. Mở `outputs\compare_round1.jpg`.

Điểm có thể tăng, đứng yên hoặc giảm. Không sửa số trong `outputs\`. Không sửa file trong `data\test\`.

## Bước 14. Viết `reports\SELECTION.md`

File này chỉ trả lời một câu: vì sao AI đưa đúng 12 ảnh này cho bạn sửa.

1. Copy `reports\SELECTION_TEMPLATE.md` thành `reports\SELECTION.md`.
2. Mở `outputs\selection_round1.csv` bằng Excel. Mỗi dòng là một ảnh. Cột `score` là điểm. Cột `selected` ghi `True` thì ảnh đó nằm trong 12 ảnh bạn sửa.
3. Xóa chữ `ĐIỀN`. Viết 4 đoạn tiếng Việt, kiểu như ví dụ dưới. Đổi tên ảnh và số điểm cho đúng file CSV.

```text
Nếu chỉ được sửa 5 ảnh, tôi chọn frame_0182.jpg (hạng 1, điểm 0.959), frame_0369.jpg (hạng 2, điểm 0.932), frame_0380.jpg (hạng 3, điểm 0.917), frame_0326.jpg (hạng 4, điểm 0.916) và frame_0331.jpg (hạng 5, điểm 0.915). Năm ảnh này đứng đầu danh sách. frame_0182.jpg và frame_0187.jpg cách nhau khoảng 2 giây, cảnh gần giống nhau, nên tôi không lấy cả hai.

Trong 12 ảnh AI đã chọn, tôi nhìn frame_0182.jpg, frame_0099.jpg và frame_0107.jpg. Cả ba đều có điểm trên 0.88 và AI khoanh nhiều xe nhưng còn nhiều khung chưa chắc.

frame_0372.jpg hạng 6, điểm 0.910, cao hơn vài ảnh đã được chọn, nhưng AI bỏ qua vì nó sát giờ với frame_0369.jpg. Hai ảnh gần như cùng một cảnh, sửa cả hai thì tốn công mà ít học thêm được gì.

Điểm cao chỉ nghĩa là AI đang phân vân. Nó chưa chứng minh sửa ảnh đó xong thì AI sẽ nhận xe tốt hơn.
```

## Bước 15. Viết `reports\REPORT.md`

File này là bài kể 5 đoạn. Copy `reports\REPORT_TEMPLATE.md` thành `reports\REPORT.md`. Ghi tên bạn. Chỗ công cụ ghi `CVAT`. Xóa chữ `ĐIỀN`, rồi viết từng mục bằng câu thường.

**Mục 1.** Trả lời: vì sao không trộn ảnh học và ảnh kiểm tra?

Viết: camera đứng một chỗ, một chiếc xe nằm trong hình vài giây. Ảnh học và ảnh kiểm tra phải cách nhau theo thời gian. Nếu trộn ngẫu nhiên, cùng một xe có thể vừa được AI học vừa được dùng để chấm. Điểm sẽ đẹp hơn sự thật.

**Mục 2.** Trả lời: lần chạy đầu AI sai chỗ nào?

Mở `reports\rounds_table.md`, chép dòng vòng 0. Số chính là điểm khớp khung `0.771`. Xe nhỏ chỉ được tìm thấy khoảng `0.182`, xe vừa `0.547`, xe lớn `0.561`. Nghĩa là xe ở xa bị bỏ sót nhiều hơn xe ở gần. Mở ảnh `outputs\compare_round0.jpg` và kể một chỗ khung AI lệch. Thêm một câu: nhãn dùng để chấm cũng do máy vẽ, chưa có người xem từng khung, nên có thể nhãn chấm sai chứ không phải AI của bạn sai.

**Mục 3.** Trả lời: AI chọn ảnh bằng cách nào?

Viết: mỗi ảnh có một điểm. Một nửa điểm là AI không chắc. Ba phần mười là AI vẽ nhiều khung còn lưỡng lự. Hai phần mười là ảnh có khác thời gian với ảnh khác. Hai ảnh trong cùng lô phải cách nhau ít nhất 2 giây, vì camera đứng yên, ảnh sát nhau gần như giống hệt. Rồi nhắc lại 3 ảnh và 1 ảnh bạn đã viết ở `SELECTION.md`. Kết bằng câu: điểm cao không có nghĩa sửa ảnh đó sẽ làm AI giỏi hơn.

**Mục 4.** Trả lời: sau khi bạn sửa khung, AI có khá hơn không?

Chờ Colab chạy xong vòng 1. Chép bảng trong `rounds_table.md`. Mở `outputs\round1_diff.md` và ghi có bao nhiêu khung bạn giữ, kéo lại, xóa, thêm. So điểm vòng 1 với `0.771`. Mở `compare_round0.jpg` và `compare_round1.jpg`, kể một xe thay đổi. Nói rõ ba việc khác nhau: mắt bạn thấy gì trong `BLIND_SCAN.md`, khung nào bạn sửa ghi trong `REVIEW_LOG.csv`, và AI sau khi học lại ra sao.

**Mục 5.** Trả lời: dừng hay làm tiếp?

Viết điểm vòng 1 so với vòng 0, rồi chọn dừng hoặc làm tiếp và nói vì sao. Nêu hai chỗ còn yếu, ví dụ xe xa chỉ còn hai chấm đèn, hoặc xe bị cắt mép. Nói sửa thêm thì mất thời gian, và không nên chọn hai ảnh sát nhau vì chúng gần như một cảnh. Nhắc: phần chấm chỉ có 20 ảnh, xe quá nhỏ không tính, nhãn chấm chưa được người kiểm. Nếu điểm giảm, hãy xem lại khung bạn đã sửa trước khi cho AI học thêm.

## Bước 16. Kiểm tra rồi nộp

Trong PowerShell, ở thư mục dự án:

```powershell
python tools/check_submission.py
```

Sửa mọi dòng lỗi nó in ra, rồi chạy lại đến khi không còn lỗi.

Mở GitHub Desktop. **Commit to main → Push origin**.

Mở repo của bạn trên GitHub và kiểm tra repo đang **Public**. Trên repo phải có:

- `labels/round1/`
- `outputs/metrics_round0.json` và `outputs/metrics_round1.json`
- `outputs/selection_round1.csv` và `outputs/selection_round1.jpg`
- `outputs/compare_round0.jpg` và `outputs/compare_round1.jpg`
- `outputs/round1_diff.json` và `outputs/round1_diff.md`
- `reports/BLIND_SCAN.md`, `reports/blind_lock.json`, `reports/REVIEW_LOG.csv`
- `reports/SELECTION.md`, `reports/rounds_table.md`, `reports/REPORT.md`

Không đưa lên GitHub: `day8_data.zip`, thư mục `to_label`, các file zip. Giữ chúng trên máy.

## Không được làm

- Không sửa `reports\BLIND_SCAN.md` sau khi đã khóa.
- Không sửa bất kỳ file nào trong `data\test\`.
- Không sửa tay các file số đo trong `outputs\`.
- Không chép nhãn hoặc số đo của người khác.
