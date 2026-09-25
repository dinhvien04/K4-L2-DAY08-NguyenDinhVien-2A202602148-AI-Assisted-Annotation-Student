# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyen Dinh Vien

Công cụ gán nhãn đã dùng: CVAT / AnyLabeling (định dạng xuất Ultralytics YOLO Detection 1.0)

---

## 1. Dữ liệu và cách chia tập

Tập chưa gán nhãn (`pool` gồm 268 ảnh) và tập kiểm thử (`test` gồm 20 ảnh) được chia theo trục thời gian kèm vùng đệm (buffer 112 ảnh) thay vì chia ngẫu nhiên vì các lý do kỹ thuật sau:

1. **Bản chất của dữ liệu video và camera cố định**: Video gốc dài 160 giây được quay từ camera đặt cố định trên cầu vượt nhìn xuống đường cao tốc, lấy mẫu ở tần số 2.5 fps (mỗi khung hình cách nhau 0.4 giây). Do camera hoàn toàn đứng yên, nền cảnh phía sau tĩnh 100%. Mỗi phương tiện khi di chuyển qua khung hình mất từ vài giây đến hơn chục giây, xuất hiện liên tiếp trên hàng chục frame kề nhau với hình thái, kích thước và màu sắc hầu như không đổi.
2. **Nguy cơ rò rỉ dữ liệu (Data Leakage)**: Nếu chia ngẫu nhiên (random split), các frame nằm sát nhau (chỉ cách nhau 0.4–0.8 giây) chắc chắn sẽ bị phân tán vào cả tập huấn luyện lẫn tập kiểm thử. Khi đó, cùng một chiếc xe vừa nằm trong tập huấn luyện vừa nằm trong tập kiểm thử. Mô hình sẽ chỉ đơn giản là "học vẹt" hoặc nhận diện lại chiếc xe mà nó đã từng nhìn thấy thay vì học được khả năng khái quát hóa (generalization) các đặc trưng xe ban đêm.
3. **Hướng lệch của số đo nếu chia ngẫu nhiên**: Số đo hiệu năng (AP50, Precision, Recall) trên tập kiểm thử sẽ bị **thổi phồng nghiêm trọng theo hướng lạc quan giả tạo (over-optimistic)**. Một mô hình có số đo rất cao khi chia ngẫu nhiên sẽ sụp đổ hoàn toàn khi triển khai thực tế trên một đoạn video mới hoặc một luồng xe khác.

Để giải quyết triệt để rò rỉ dữ liệu, bài thực hành đã chia tập kiểm thử thành 4 đoạn độc lập có tâm tại các mốc giây 20, 60, 100 và 140 (mỗi đoạn 5 ảnh cách nhau 1.2s, tổng 20 ảnh test). Giữa các đoạn test và tập pool được đặt **vùng đệm 112 ảnh (khoảng 4 giây trước và sau mỗi đoạn test)** bị loại bỏ hoàn toàn. Nhờ vậy, ảnh pool gần ảnh kiểm thử nhất vẫn cách nhau ít nhất **4.4 giây**, đảm bảo mọi chiếc xe trong tập test là hoàn toàn mới đối với mô hình huấn luyện.

---

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng kết quả vòng 0 từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Chi tiết từ `outputs/metrics_round0.json` và quan sát trên `outputs/compare_round0.jpg`:
- **Đánh giá tổng quan**: Mô hình khởi đầu lạnh `yolov8n` (pretrained trên COCO gộp 3 lớp car, bus, truck thành `car`) đạt AP50 = 0.771 (0.7714), Precision@0.25 đạt 0.925 (0.9249). Tuy nhiên, độ phủ Recall@0.25 rất thấp, chỉ đạt **0.489 (48.88%)** với F1 = 0.640. Trong tổng số 403 box tham chiếu (đã bỏ qua 14 box quá nhỏ < 16px), mô hình chỉ đạt 197 True Positives (TP), dính 16 False Positives (FP) và bỏ sót tới **206 False Negatives (FN)**.
- **Không khớp nhãn tham chiếu ở những loại xe nào**:
  1. *Xe nhỏ ở xa*: Nhóm xe này bị bỏ sót nhiều nhất. Recall của xe kích thước nhỏ (`small`, 66 box tham chiếu) chỉ đạt **0.182 (18.18%)**, tức bỏ sót hơn 80% xe nhỏ. Xe ở xa ban đêm chỉ hiển thị như hai chấm sáng nhỏ (đèn pha trắng hoặc đèn hậu đỏ), thiếu hẳn cấu trúc hình học thân xe mà mô hình COCO ban ngày thường dựa vào.
  2. *Xe bị bóng tối che khuất hoặc xe màu tối ở làn giữa*: Các xe chạy làn giữa không bật đèn rọi vào thân xe khác thường bị chìm vào nền đường tối, mô hình bỏ qua hoàn toàn.
  3. *Lỗi phát hiện nhầm (FP = 16)*: Mô hình bắt nhầm các vệt sáng đèn pha phản chiếu trên mặt đường ướt (như thấy rõ ở `frame_0350` và `frame_0150` trong `compare_round0.jpg`) hoặc nhầm các cụm đèn biển báo phản quang thành phương tiện.
- **Trường hợp cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai**:
  Nhãn tham chiếu của tập test (`data/test/labels/`) được tạo tự động bởi mô hình phát hiện đối tượng khác và **chưa từng được con người rà soát thủ công từng box**. Ví dụ trên `frame_0350`, có những vùng phản quang hoặc đốm đèn ở rất xa sát chân trời mà nhãn tham chiếu vẫn gắn box (hoặc ngược lại, có xe con thực tế chạy ở làn biên nhưng nhãn tham chiếu lại bỏ sót). Nếu mô hình dự đoán đúng xe thật mà nhãn tham chiếu không có, mô hình bị phạt oan lỗi False Positive; ngược lại nếu nhãn tham chiếu vẽ box vào bóng tối không có xe, mô hình bị phạt oan lỗi False Negative. Do đó, cần kiểm tra trực quan ảnh gốc trước khi khẳng định mô hình dự đoán sai.

---

## 3. Chiến lược chọn mẫu

### 3.1. Giải thích công thức và vai trò của `MIN_GAP_S`
Công thức tính điểm ưu tiên chọn mẫu:
$$\text{score} = W_U \cdot U + W_A \cdot A + W_D \cdot D = 0.5 \cdot U + 0.3 \cdot A + 0.2 \cdot D$$

- **$U$ (Uncertainty - trọng số 0.5)**: Đo lường độ bất định của các bounding box do mô hình đề xuất trên frame. Điểm $U$ càng cao chứng tỏ mô hình càng thiếu tự tin (phân vân về xác suất phân lớp quanh ngưỡng quyết định). Chiếm 50% trọng số vì đây là tín hiệu cốt lõi của Active Learning.
- **$A$ (Ambiguity - trọng số 0.3)**: Đo tỷ lệ và số lượng các box rơi vào "vùng mơ hồ" (confidence nằm trong khoảng 0.25 đến 0.50). Frame có nhiều box mơ hồ là frame mà AI do dự nhiều nhất giữa việc giữ hay bỏ, rất cần sự thẩm định của con người.
- **$D$ (Diversity - trọng số 0.2)**: Đại diện cho độ phân tán theo trục thời gian, giúp trải đều các mẫu được chọn qua toàn bộ video, tránh tập trung cục bộ.
- **Vai trò của `MIN_GAP_S = 2.0s`**: Đây là ràng buộc khoảng cách thời gian tối thiểu giữa hai ảnh được chọn vào cùng một lô. Vì camera cố định và xe chạy trên đường cao tốc, hai ảnh cách nhau dưới 2 giây có góc nhìn, bối cảnh và các phương tiện gần như giống hệt nhau (near-duplicates). Ràng buộc `MIN_GAP_S` đóng vai trò cơ chế ức chế phi cực đại (Non-Maximum Suppression theo thời gian), ngăn chặn việc lãng phí ngân sách gán nhãn vào các frame trùng lặp.

### 3.2. Dẫn chứng 3 frame model chọn và 1 frame khác
1. **`frame_0182.jpg` (Rank 1, t_sec: 72.8s, Score: 0.9591, U: 0.9182, A: 1.0000)**: Mật độ xe lớn ở cả hai chiều, số box mơ hồ cao nhất pool (`n_ambiguous: 18`), chi phí gán nhãn vừa phải (28 box đề xuất), giá trị thông tin mang lại tối đa.
2. **`frame_0369.jpg` (Rank 2, t_sec: 147.6s, Score: 0.9324, U: 0.9315, A: 0.8889)**: Đại diện cho phân phối giao thông ở cuối video (t=147.6s), điểm bất định cực cao (`U = 0.9315`, 43 box đề xuất).
3. **`frame_0331.jpg` (Rank 5, t_sec: 132.4s, Score: 0.9154, U: 0.8308, A: 1.0000)**: Chứa dòng xe lớn phức tạp nhất với 47 box đề xuất (thực tế rà soát có 37 box và 7 box bị xóa do bắt nhầm vệt đèn), thể hiện rõ chi phí gán nhãn rất lớn nhưng có nhiều ca khó.
4. **Frame bị loại bỏ do gần trùng**: **`frame_0372.jpg` (Rank 6, Score: 0.9101, t_sec: 148.8s)** có điểm số cao thứ 6 toàn pool, nhưng bị gạt bỏ (`selected = False`) vì nằm cách `frame_0369.jpg` (t=147.6s) chỉ 1.2 giây ($< 2.0s$). Chọn cả hai sẽ gây lãng phí công rà nhãn cho cùng một tập xe đang di chuyển.

### 3.3. Điểm bất định có chứng minh ảnh đó sẽ cải thiện mô hình không?
**Không**. Điểm bất định chỉ là thước đo tương đối cho thấy mô hình hiện tại gặp khó khăn, chứ không đảm bảo việc gán nhãn ảnh đó sẽ nâng cao chất lượng mô hình sau fine-tune. Độ bất định có thể xuất phát từ **nhiễu ngẫu nhiên không thể loại bỏ (aleatoric uncertainty)** như vệt đèn chói lòa, sương mù, mặt đường ướt phản quang hoặc xe quá xa bị nhòe pixel. Khi ép mô hình học trên các frame quá nhiều nhiễu với một tập train siêu nhỏ (12 ảnh), mô hình rất dễ bị "rối loạn" phân phối trọng số, dẫn đến hiện tượng quá khắt khe hoặc suy giảm độ phủ trên tập test.

---

## 4. Các vòng học chủ động (active learning)

Bảng tổng hợp từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 340 | 0.612 | -0.159 | 1.000 | 0.218 | 0.358 | 0.000 | 0.277 | 0.146 |

### 4.1. Mức độ sửa nhãn gợi ý vòng 1 (từ `outputs/round1_diff.md`)
Trên 12 ảnh của lô 1, mô hình đề xuất 169 box, sau khi con người rà soát và chỉnh sửa đã thu được **340 box** (tăng hơn gấp đôi):
- **accepted (giữ nguyên)**: 78 box (tỷ lệ chấp nhận đạt 46.15%).
- **edited (chỉnh sửa)**: 69 box (chủ yếu là co box lại để ôm sát thân xe, cắt bỏ phần vệt đèn pha rọi trên đường).
- **deleted (xóa bỏ, FP của model)**: 22 box (loại bỏ các box AI nhận nhầm vệt phản chiếu mặt đường hoặc các box vẽ trùng nhau, ví dụ `frame_0331.jpg` có tới 7 box bị xóa).
- **added (thêm mới, FN của model)**: 193 box (bổ sung số lượng cực lớn các xe bị AI bỏ sót, đặc biệt là xe ở làn xa và xe con tối màu).

### 4.2. Biến động AP50 và các nhóm xe
- **Biến động AP50**: AP50 giảm từ **0.771 xuống 0.612**, tức $\Delta\text{AP50} = -0.159$ (-0.1594).
- **Precision**: Tăng tuyệt đối từ **0.925 lên 1.000 (100%)**, số ca False Positive giảm hoàn toàn từ 16 về **0**.
- **Recall**: Sụt giảm nghiêm trọng từ **0.489 xuống 0.218** (số FN tăng vọt từ 206 lên 315).
  - Xe nhỏ (`small`): Recall giảm từ 0.182 về **0.000** (không nhận diện được bất kỳ xe nhỏ nào trên ngưỡng conf 0.25).
  - Xe vừa (`medium`): Recall giảm từ 0.547 xuống **0.277**.
  - Xe lớn (`large`): Recall giảm từ 0.561 xuống **0.146**.

### 4.3. Phân tích ca thay đổi trên `compare_round1.jpg` và giả thuyết kỹ thuật
- **Quan sát ca cụ thể**:
  - Tại `frame_0050`: Ở cold start, model đạt TP 11, FP 2, FN 7. Sang Round 1, model đạt **TP 3, FP 0, FN 15**.
  - Tại `frame_0150`: Ở cold start, model đạt TP 10, FP 2, FN 10. Sang Round 1, model đạt **TP 4, FP 0, FN 16**.
- **Giả thuyết có căn cứ kỹ thuật (không khẳng định tuyệt đối)**:
  1. *Độ chính xác tăng tuyệt đối (FP = 0)*: Người gán nhãn đã xóa triệt để 22 box giả và cắt gọn 69 box vệt đèn pha trong tập train. Mô hình đã học rất tốt việc "không được bắt nhầm vệt đèn" và "không vẽ box bừa bãi", dẫn tới Precision đạt 1.000.
  2. *Độ phủ giảm mạnh (Recall giảm)*: Do kích thước tập huấn luyện quá nhỏ (chỉ 12 ảnh với 340 box) nhưng lại huấn luyện trong 50 epochs bắt đầu từ checkpoint `yolov8n.pt`, mô hình có thể đã trải qua hiện tượng **dịch chuyển phân bố độ tự tin (confidence calibration shift)**. Mô hình trở nên quá dè dặt (under-confident) với các vật thể ở xa hoặc ánh sáng yếu; điểm confidence của các xe này bị đẩy xuống dưới ngưỡng lọc mặc định `conf_thr = 0.25`, dẫn đến bị tính là FN dù có thể box dự đoán vẫn tồn tại ở ngưỡng thấp hơn.

### 4.4. Đối chiếu giữa Blind Scan, Review Log và Kết quả Model
- **Quan sát độc lập (`BLIND_SCAN.md` trên `frame_0099.jpg`)**: Người quét độc lập đếm thấy 24 xe bằng mắt thường, dự báo AI sẽ bắt nhầm vệt đèn pha rọi dài xuống mặt đường ướt và bỏ sót các xe nhỏ phía xa.
- **Lỗi pre-label đã sửa (`round1_diff.md` & `REVIEW_LOG.csv`)**: Thực tế AI chỉ đề xuất 13 box trên `frame_0099.jpg`. Người đã thêm 11 box xe bị sót, chỉnh sửa 7 box bị phình to do dính vệt đèn pha và giữ nguyên 6 box đúng, nâng tổng số box lên 24. Trong `REVIEW_LOG.csv`, các ca chỉnh sửa và thêm mới này được ghi nhận cụ thể theo đúng `GUIDELINE_LABEL.md`.
- **Kết quả model sau fine-tune**: Model khắc phục hoàn hảo việc phát hiện nhầm vệt sáng (Precision đạt 1.000 trên cả 20 ảnh test), nhưng lại sinh ra hiệu ứng phụ là quá khắt khe, bỏ sót nhiều xe nhỏ/xa khiến Recall giảm.
- **Ca khó theo guideline**: Xe con tối màu chạy ngược chiều ở làn trái (`frame_0099.jpg`): Đèn pha rọi vệt sáng rất mạnh xuống mặt đường ướt trong khi thân xe màu đen chìm hoàn toàn vào bóng đêm. Quy tắc yêu cầu chỉ vẽ box ôm sát thân xe đoán được, loại bỏ vệt sáng đèn pha. Đây là tình huống khó đòi hỏi người gán nhãn phải ước lượng hình học xe dựa trên vị trí cụm đèn pha.

---

## 5. Kết luận và giới hạn

### 5.1. Đánh giá kết quả và quyết định dừng/tiếp tục
- **Kết quả**: Vòng 1 ghi nhận bước tiến lớn về độ sạch của nhãn (Precision đạt 1.000, sạch bóng False Positive), nhưng AP50 sụt giảm (-0.159) do Recall giảm sâu, đặc biệt ở nhóm xe nhỏ (Recall = 0.000).
- **Quyết định**: **DỪNG LẠI, KHÔNG LÀM TIẾP VÒNG 2**.
  - *Lý do*: Sự suy giảm AP50 không bắt nguồn từ việc thiếu số lượng ảnh dán nhãn, mà xuất phát từ việc cấu hình huấn luyện (50 epochs trên 12 ảnh với learning rate mặc định) gây co cụm độ tự tin (under-confidence). Nếu tiếp tục gán nhãn thêm 12 ảnh ở vòng 2 mà không tinh chỉnh siêu tham số (như learning rate, freeze backbone, hoặc hạ ngưỡng confidence khi đánh giá), ta sẽ tiếp tục tiêu tốn công sức vô ích mà không giải quyết được gốc rễ bài toán.

### 5.2. Đề xuất hai ca còn yếu/bất định cho vòng sau (nếu tiếp tục)
1. **Ca 1 - Xe kích thước nhỏ ở cự ly xa (`small cars`)**: Cần chọn các frame có mật độ xe nhỏ cao ở phía chân trời để model học lại đặc trưng nhận dạng xe qua cụm đèn trong đêm, giải quyết mức recall 0.000 hiện tại.
2. **Ca 2 - Xe buýt và phương tiện lớn có nguồn sáng phức tạp** (như `frame_0392.jpg` với biển quảng cáo phát sáng, $U = 0.9747$): Giúp mô hình phân biệt tốt thân xe lớn với ánh sáng phát ra từ chính nó.
- *Chi phí gán nhãn và nguy cơ ảnh gần trùng*: Các frame này có số lượng box rất lớn (35–45 box), chi phí gán nhãn cao gấp 2–3 lần frame thông thường. Nếu không tuân thủ nghiêm ngặt ràng buộc `min_gap_s >= 2.0s`, rất dễ chọn phải các ảnh gần trùng, gây lãng phí tài nguyên mà không tăng độ đa dạng dữ liệu.

### 5.3. Các giới hạn ảnh hưởng đến kết luận
1. **Tập kiểm thử chỉ có 20 ảnh**: Kích thước mẫu quá nhỏ khiến các chỉ số thống kê có phương sai lớn. Một vài box dao động quanh ngưỡng IoU 0.5 hoặc conf 0.25 có thể làm AP50 biến động mạnh.
2. **Quy tắc bỏ qua xe nhỏ < 16 pixel**: Có 14/417 box tham chiếu bị loại bỏ khi đánh giá, khiến việc đo lường độ phủ thực sự ở cự ly rất xa bị giới hạn.
3. **Nhãn tham chiếu do mô hình tự động tạo ra**: Nhãn test chưa được con người thẩm định, do đó AP50 chỉ phản ánh độ khớp với mô hình sinh nhãn tham chiếu chứ không phải chân lý tuyệt đối (ground truth thực tế). Việc Precision đạt 1.000 nhưng AP50 giảm có thể một phần do mô hình sau fine-tune từ chối dự đoán các box nhiễu mà nhãn tham chiếu vô tình ghi nhận.

### 5.4. Kế hoạch kiểm tra trước khi train thêm (nếu tiếp tục)
Nếu muốn tiếp tục huấn luyện trong tương lai để cải thiện AP50, tôi sẽ thực hiện các bước kiểm tra theo thứ tự:
1. **Phân tích phân phối Confidence Score**: Xuất biểu đồ phân bố confidence của các box dự đoán trên tập test để kiểm tra xem liệu model có dự đoán đúng xe nhưng điểm confidence bị rơi vào khoảng 0.15–0.24 (dưới ngưỡng lọc 0.25) hay không.
2. **Kiểm tra trực quan các ca FN trên ảnh test**: Đối chiếu các box FN trên `compare_round1.jpg` với ảnh gốc để xác định đó là xe thật bị sót hay là do nhãn tham chiếu bị lỗi.
3. **Rà soát chất lượng bộ nhãn huấn luyện**: Đảm bảo toàn bộ 340 box trong `labels/round1/` không bị bỏ sót các xe nhỏ ở xa một cách có hệ thống.
4. **Điều chỉnh chiến lược huấn luyện**: Giảm số epoch (ví dụ xuống 20–30 epochs), đóng băng các tầng trích xuất đặc trưng của backbone (`freeze=10`), và sử dụng kỹ thuật fine-tuning bảo toàn tri thức cũ (learning rate nhỏ hơn) nhằm tránh hiện tượng catastrophic forgetting hoặc under-confidence.
