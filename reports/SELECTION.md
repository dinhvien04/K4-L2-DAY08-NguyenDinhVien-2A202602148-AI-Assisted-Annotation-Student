# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, nếu chỉ có ngân sách rà soát 5 ảnh (thay vì 12 ảnh), tôi sẽ ưu tiên 5 frame sau nhằm tối ưu hóa giá trị thông tin thu nhận, trải rộng phân bố thời gian (diversity) và kiểm soát chi phí gán nhãn:

1. **`frame_0182.jpg` (Rank 1, t_sec: 72.8s, Score: 0.9591, U: 0.9182, A: 1.0000, D: 1.0000)**:
   - *Lý do*: Đứng đầu bảng xếp hạng toàn bộ pool. Frame có số lượng box mơ hồ cao nhất (`n_ambiguous: 18`, `A: 1.0`), độ bất định rất cao (`U: 0.9182`). Số lượng box mô hình đề xuất ở mức vừa phải (28 box), giúp chi phí rà nhãn không quá nặng nề nhưng mang lại giá trị định hình ranh giới xe lớn nhất cho mô hình.
2. **`frame_0369.jpg` (Rank 2, t_sec: 147.6s, Score: 0.9324, U: 0.9315, A: 0.8889, D: 1.0000)**:
   - *Lý do*: Nằm ở giai đoạn cuối của video (t=147.6s), tạo khoảng cách thời gian lớn (hơn 74 giây) so với `frame_0182.jpg`, đảm bảo tính đa dạng tuyệt đối về luồng giao thông. Mặc dù số lượng box cao (43 box) làm tăng chi phí gán nhãn, nhưng điểm bất định `U = 0.9315` và 16 box mơ hồ phản ánh nhiều ca khó cần con người gán nhãn chuẩn hóa.
3. **`frame_0326.jpg` (Rank 4, t_sec: 130.4s, Score: 0.9155, U: 0.9310, A: 0.8333, D: 1.0000)**:
   - *Lý do*: Thuộc cụm thời gian 130s. Quyết định quan trọng ở đây là **chọn `frame_0326.jpg` và bỏ qua `frame_0331.jpg` (Rank 5, t=132.4s, Score: 0.9154)**. Hai ảnh này chỉ cách nhau 2.0 giây, cảnh quay gần như trùng lặp (near-duplicates) với cùng một luồng xe tải và xe buýt. Hơn nữa, `frame_0331.jpg` có tới 47 box đề xuất (thực tế sau khi rà soát phải xử lý tới 37 box và xóa bỏ 7 box lỗi), chi phí rà nhãn quá tốn kém. Việc chọn `frame_0326.jpg` (39 box) giúp tiết kiệm đáng kể thời gian gán nhãn mà vẫn nắm bắt được đặc trưng của dòng xe lớn ở đoạn này.
4. **`frame_0099.jpg` (Rank 8, t_sec: 39.6s, Score: 0.9063, U: 0.9460, A: 0.7778, D: 1.0000)**:
   - *Lý do*: Bổ sung mẫu đại diện cho giai đoạn đầu video (t=39.6s, trước cả cụm 72.8s). Điểm bất định `U = 0.9460` thuộc nhóm cao nhất, trong khi số box chỉ là 29 box (`n_ambiguous: 14`), mang lại tỷ suất thông tin trên chi phí gán nhãn (information per annotation cost) rất cao.
5. **`frame_0227.jpg` (Rank 11, t_sec: 90.8s, Score: 0.8915, U: 0.9164, A: 0.7778, D: 1.0000)**:
   - *Lý do*: Nằm ở mốc 90.8s, lấp đầy khoảng trống thời gian giữa `frame_0182.jpg` (72.8s) và `frame_0326.jpg` (130.4s). Điểm bất định cao (`U = 0.9164`), 37 box với 14 box mơ hồ, phản ánh cảnh xe tải chạy làn giữa trong đêm với ánh đèn pha rọi ngược.

---

### Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV / ảnh contact sheet (`selection_round1.jpg`):

1. **`frame_0182.jpg` (Rank 1, t_sec: 72.8s, Score: 0.9591)**:
   - *Bằng chứng*: Trên contact sheet, ảnh hiển thị mật độ xe dày đặc ở cả làn ngược chiều lẫn cùng chiều. Đèn pha các xe con ở làn giữa chiếu vệt sáng kéo dài trên mặt đường ướt. CSV ghi nhận `U = 0.9182`, `A = 1.0000` (mức tối đa với 18 box mơ hồ), thể hiện mô hình có rất nhiều box rơi vào khoảng mập mờ $0.15 \le \text{confidence} < 0.50$.
2. **`frame_0331.jpg` (Rank 5, t_sec: 132.4s, Score: 0.9154)**:
   - *Bằng chứng*: Trên contact sheet, đây là frame có số lượng phương tiện lớn và phức tạp nhất với các xe tải lớn, xe khách và dòng xe con nối đuôi. CSV chứng minh đây là ảnh có số box đề xuất cao nhất trong lô (`n_boxes: 47`, `n_ambiguous: 18`, `A: 1.0000`).
3. **`frame_0392.jpg` (Rank 15, t_sec: 156.8s, Score: 0.8874)**:
   - *Bằng chứng*: Trên contact sheet, frame này nổi bật với hai chiếc xe buýt lớn có biển quảng cáo phát sáng bên hông ở làn giữa, cùng nhiều ánh đèn pha chói lọi từ chiều ngược lại. Trong CSV, frame này có điểm bất định cá thể cao nhất toàn bộ lô (`U = 0.9747`), chứng minh mô hình cold start bị nhiễu loạn nghiêm trọng bởi ánh sáng phức tạp từ thân xe buýt và đèn đường.

---

### Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:

- **Frame điểm cao nhưng không được chọn**: **`frame_0372.jpg` (Rank 6, Score: 0.9101, t_sec: 148.8s, U: 0.9202, A: 0.8333, n_boxes: 42)**.
  - *Lý do*: Mặc dù có điểm số xếp thứ 6 trong toàn bộ 268 ảnh ứng viên của pool (cao hơn nhiều frame được chọn như Rank 7, 8, 10, 11, 13, 14, 15), frame này đã bị thuật toán Active Learning gạt bỏ (`selected = False`). Nguyên nhân là do ràng buộc đa dạng thời gian `MIN_GAP_S = 2.0s`. Trước đó, `frame_0369.jpg` (Rank 2, t=147.6s) đã được chọn, và khoảng cách thời gian giữa hai ảnh chỉ là $|148.8 - 147.6| = 1.2s < 2.0s$. Vì camera đặt cố định trên cầu vượt, các xe di chuyển trên cao tốc trong 1.2 giây gần như giữ nguyên vị trí và hình thái (near-duplicates). Nếu chọn cả hai, ta sẽ lãng phí chi phí rà nhãn (42 box) cho cùng một ngữ cảnh mà không cung cấp thêm tri thức mới cho mô hình. (Trường hợp tương tự xảy ra với `frame_0368.jpg` Rank 9 cách 0.4s và `frame_0330.jpg` Rank 12 cách 0.4s).

---

### Điều phép chọn này chưa chứng minh về chất lượng mô hình:

1. **Điểm bất định ($U, A$) không đồng nghĩa với khả năng cải thiện mô hình**: Điểm số chỉ phản ánh trạng thái "bối rối" của mô hình hiện tại trước các mẫu dữ liệu, nhưng không thể phân biệt giữa độ bất định do thiếu dữ liệu học (epistemic uncertainty - có thể cải thiện khi gán nhãn) và độ bất định do nhiễu vật lý không thể tránh khỏi (aleatoric uncertainty - như chói lóa đèn pha, mặt đường phản chiếu, xe quá xa nhòe mờ). Nếu chọn phải các frame chứa quá nhiều nhiễu ngẫu nhiên, mô hình có thể bị quá khớp (overfitting) hoặc học sai phân phối.
2. **Chưa chứng minh được năng lực tổng quát hóa trên tập kiểm thử độc lập**: Phép chọn hoàn toàn thực hiện heuristic trên tập pool chưa gán nhãn. Nó không bảo đảm rằng việc tinh chỉnh trên các ảnh này sẽ giúp mô hình tăng chỉ số AP50 hay Recall trên tập test, đặc biệt là khi tập test có phân phối khác hoặc khi bộ nhãn tham chiếu có những đặc điểm riêng.
