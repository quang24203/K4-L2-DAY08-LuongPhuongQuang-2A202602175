# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét ảnh gần trùng hoặc trường hợp model không dự đoán được box:

Nếu chỉ có ngân sách gán nhãn cho 5 ảnh, tôi đề xuất ưu tiên 5 frame sau:
1. **`frame_0182.jpg`** (Thứ tự: Rank 1, Thời điểm: 72.8s, Điểm tổng: 0.9591, U=0.9182, A=1.0000, D=1.0000, 28 box, 18 box mơ hồ): Frame có điểm cao nhất toàn bộ 268 ảnh pool; số lượng box mơ hồ đạt mức tối đa (A=1.0 với 18 box có confidence trong khoảng [0.15, 0.50]), thể hiện sự phân vân cực lớn của mô hình; nằm ở đoạn giữa video (72.8s) khi mật độ xe bắt đầu tăng.
2. **`frame_0369.jpg`** (Thứ tự: Rank 2, Thời điểm: 147.6s, Điểm tổng: 0.9324, U=0.9315, A=0.8889, D=1.0000, 43 box, 16 box mơ hồ): Đại diện cho đoạn cuối video với mật độ lưu thông cao (43 xe); độ bất định của 5 box khó nhất rất cao (U=0.9315), nhiều tình huống xe che khuất lẫn nhau và xe ở xa cần sự can thiệp chuẩn hóa từ người rà nhãn.
3. **`frame_0326.jpg`** (Thứ tự: Rank 4, Thời điểm: 130.4s, Điểm tổng: 0.9155, U=0.9310, A=0.8333, D=1.0000, 39 box, 15 box mơ hồ): Cung cấp các tình huống luồng xe chuyển làn phức tạp. Quyết định chọn frame này đi kèm việc **loại bỏ** các ứng viên rank cao kế tiếp như `frame_0331.jpg` (Rank 5, t=132.4s, cách 2.0s), `frame_0330.jpg` (Rank 12, t=132.0s, cách 1.6s) và `frame_0328.jpg` (Rank 26, t=131.2s, cách 0.8s) vì trùng lặp cảnh gần. Do camera cố định, các frame cách nhau dưới 2 giây chứa hầu như cùng một đoàn xe; trong điều kiện ngân sách hạn hẹp 5 ảnh, việc loại bỏ ảnh trùng cảnh là bắt buộc để tránh lãng phí chi phí gán nhãn.
4. **`frame_0099.jpg`** (Thứ tự: Rank 8, Thời điểm: 39.6s, Điểm tổng: 0.9063, U=0.9460, A=0.7778, D=1.0000, 29 box, 14 box mơ hồ): Đại diện tiêu biểu cho đoạn đầu video (t=39.6s). Ảnh có độ bất định của 5 box khó nhất cao vượt trội (U=0.9460, cao nhất trong top 10). Nếu chỉ chọn thuần theo điểm rank 1 đến 5, các ảnh sẽ bị dồn cục bộ về nửa sau video (t > 72s đến 150s). Chọn frame_0099 giúp bảo đảm tính đa dạng thời gian (temporal diversity) và phân bố dữ liệu đồng đều.
5. **`frame_0227.jpg`** (Thứ tự: Rank 11, Thời điểm: 90.8s, Điểm tổng: 0.8915, U=0.9164, A=0.7778, D=1.0000, 37 box, 14 box mơ hồ): Lấp khoảng trống thời gian lớn giữa 72.8s và 130.4s. Điểm U cao (0.9164), 37 xe lưu thông, hỗ trợ mô hình nhận diện tốt hơn các xe ở làn giữa và làn xa trong giai đoạn này.

---

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:
- **`frame_0182.jpg`**: Đứng vị trí Rank 1 trong `outputs/selection_round1.csv` (score=0.9591, U=0.9182, A=1.0000, n_ambiguous=18). Trên ảnh contact sheet `outputs/selection_round1.jpg`, frame này xuất hiện đầu tiên, cho thấy mật độ xe lưu thông dàn trải, AI bị phân vân mạnh ở các xe tối màu làn giữa và xe tải lớn ở góc dưới (thực tế người đã thêm 14 box FN).
- **`frame_0331.jpg`**: Đứng vị trí Rank 5 trong CSV (score=0.9154, U=0.8308, A=1.0000, n_boxes=47, n_ambiguous=18). Đây là một trong những frame có số lượng xe đông nhất lô. Trên contact sheet, ảnh thể hiện rõ sự nhiễu loạn do ánh đèn pha rọi xuống mặt đường, khiến AI tạo ra các box ảo và box chồng lấn (thực tế khi rà nhãn đã xóa 5 box FP và sửa 4 box).
- **`frame_0099.jpg`**: Đứng vị trí Rank 8 trong CSV (score=0.9063, U=0.9460, A=0.7778, n_boxes=29). Điểm U đạt 0.9460 là mức bất định cá thể cao nhất. Trên contact sheet, có thể thấy rõ các xe bị cắt nửa thân ở mép trái và mép phải khung hình (đúng như đã ghi nhận trong `reports/BLIND_SCAN.md`), nơi AI bỏ sót hoàn toàn hoặc vẽ box co cụm chỉ vào cụm đèn sau.

---

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:
- **Frame có điểm cao nhưng không chọn**: `frame_0372.jpg` (Rank 6 trong `outputs/selection_round1.csv`, score=0.9101, U=0.9202, A=0.8333, D=1.0000, n_boxes=42). Mặc dù điểm tổng đứng thứ 6 và vượt trội nhiều ảnh được chọn, frame này bị loại bởi thuật toán vì thời điểm xuất hiện là `t = 148.8s`, chỉ cách `frame_0369.jpg` (Rank 2, `t = 147.6s`) đúng 1.2 giây (nhỏ hơn ngưỡng `MIN_GAP_S = 2.0s`). Do camera gắn cố định trên cầu vượt, hai khung hình cách nhau 1.2s có nền cảnh và dàn xe gần như y hệt; việc gán nhãn cả hai sẽ gây trùng lặp thông tin và lãng phí công sức gán nhãn. Tương tự, `frame_0368.jpg` (Rank 9, t=147.2s, score=0.9003) cũng bị thuật toán lọc bỏ vì chỉ cách frame_0369 đúng 0.4s.
- **Frame có điểm thấp vẫn nên xem**: `frame_0002.jpg` (Rank 18, t=0.8s, score=0.8658, U=0.8983). Đây là khung hình ở ngay đầu video khi camera vừa bắt đầu ghi hình. Điểm score thấp hơn do số lượng box phát hiện ít hơn (27 box), nhưng lại chứa các ca biên đặc biệt như xe bắt đầu tiến vào tầm nhìn từ xa trong điều kiện ánh sáng sớm của cảnh quay.

---

Điều phép chọn này chưa chứng minh về chất lượng mô hình:
- Điểm bất định cao (U và A cao) chỉ phản ánh trạng thái thiếu tự tin của mô hình hiện tại (các dự đoán dao động quanh confidence 0.5), **hoàn toàn không chứng minh** rằng việc đưa các frame này vào huấn luyện sẽ chắc chắn giúp tăng điểm đánh giá tổng thể (AP50) trên tập kiểm thử.
- Nếu độ bất định xuất phát từ các trường hợp nhiễu không thể phân định rõ ràng (vệt phản quang trên mặt đường, quầng sáng đèn pha, xe quá nhỏ sát đường chân trời dưới 16 pixel), việc ép mô hình học có thể gây ra hiện tượng học vẹt hoặc khiến mô hình phản ứng quá dè dặt (tăng precision nhưng sụt giảm mạnh recall).
- Hơn nữa, phép chọn này thực hiện trên tập pool, trong khi tập kiểm thử (test) chỉ gồm 20 ảnh với bộ nhãn tham chiếu tự động tạo ra từ mô hình khác; do đó điểm số trên pool không đại diện cho chất lượng suy luận thực địa hay độ bao phủ thực tế trên mọi miền dữ liệu.
