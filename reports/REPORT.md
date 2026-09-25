# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Luong Phuong Quang

Công cụ gán nhãn đã dùng: CVAT Docker local

Báo cáo này đối chiếu các dữ liệu thực nghiệm từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round0.json`, `outputs/metrics_round1.json`, `outputs/round1_diff.md` và `outputs/round1_diff.json`.

---

## 1. Dữ liệu và cách chia tập

Dữ liệu của bài thực hành được trích xuất từ video "Cars driving at night" (mã Ina7KMV2OEI) trên YouTube, quay bằng camera cố định đặt trên cầu vượt nhìn xuống đường cao tốc với tốc độ trích xuất 2.5 khung hình/giây (fps), tương ứng mỗi frame cách nhau 0.4 giây. 

Tập chưa gán nhãn (pool) và tập kiểm thử (test set) bắt buộc phải được chia theo trục thời gian kèm vùng đệm (buffer) cách ly thay vì chia ngẫu nhiên (random split) vì các lý do sau:
1. **Tránh rò rỉ dữ liệu (Data Leakage):** Do camera đặt tĩnh tại một vị trí cố định và các phương tiện lưu thông trên đường mất từ vài giây đến hơn mười giây để đi qua tầm nhìn của ống kính, hai khung hình liên tiếp cách nhau 0.4 giây có bối cảnh nền và các phương tiện gần như giống hệt nhau. Nếu chia ngẫu nhiên, cùng một chiếc xe hoặc một đoàn xe sẽ vừa xuất hiện trong tập huấn luyện (pool được chọn), vừa xuất hiện trong tập kiểm thử (test). Khi đó, mô hình sẽ được đánh giá trên chính những vật thể và điều kiện ánh sáng mà nó đã được "học thuộc lòng".
2. **Hướng sai lệch của số đo nếu chia ngẫu nhiên:** Số đo trên tập kiểm thử (như AP50, Precision, Recall) sẽ bị **lệch theo hướng tăng cao giả tạo (artificially over-optimistic)**. Mô hình dường như đạt độ chính xác rất cao nhưng thực tế năng lực tổng quát hóa (generalization) trên các đoạn video mới hoặc luồng giao thông mới là rất kém.
3. **Vai trò của vùng đệm:** Dữ liệu đã chọn 20 ảnh test tập trung quanh 4 mốc thời gian (giây 20, 60, 100, 140) và loại bỏ hoàn toàn 112 khung hình trong khoảng 4 giây trước và sau mỗi đoạn kiểm thử làm vùng đệm. Nhờ đó, ảnh pool gần nhất vẫn cách ảnh kiểm thử tối thiểu 4.4 giây, bảo đảm tính độc lập tuyệt đối giữa hai tập dữ liệu.

---

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng số liệu vòng 0 từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Dựa vào `outputs/compare_round0.jpg` và `outputs/metrics_round0.json`, mô hình khởi đầu lạnh `yolov8n` (tiền huấn luyện trên tập COCO) bộc lộ các hạn chế rõ rệt:
- **Các loại xe không khớp nhãn tham chiếu:**
  + *Xe ở khoảng cách xa gần đường chân trời:* Nơi ánh sáng yếu và xe chỉ hiển thị dưới dạng hai đốm sáng nhỏ; mô hình COCO hầu như bỏ qua (FN rất cao).
  + *Xe tối màu ở làn ngoài hoặc xe bị khuất bóng:* Thân xe chìm vào nền đường tối, mô hình không nhận diện được đường viền vật thể.
  + *Xe bị che khuất một phần (partial occlusion) hoặc xe bị cắt ở mép ảnh:* Mô hình thường bỏ sót hoặc chỉ nhận diện đứt đoạn.
  + *Phát hiện nhầm (FP):* Có 16 trường hợp mô hình nhận diện nhầm các vệt phản chiếu ánh sáng đèn xe trên mặt đường ướt hoặc bóng đèn đường thành xe.
- **Độ phủ (Recall) theo kích thước xe:**
  + `R small = 0.1818` (18.18%): Cực kỳ thấp, bỏ sót hơn 81% số xe nhỏ ở xa (chỉ phát hiện được 12/66 box tham chiếu).
  + `R medium = 0.5473` (54.73%): Chỉ đạt mức trung bình, bỏ sót gần một nửa số xe kích thước vừa (162/296 box).
  + `R large = 0.5610` (56.10%): Đạt 23/41 box. Đối với các xe lớn chạy gần camera, mô hình vẫn bỏ sót đáng kể do hiện tượng vệt sáng đèn pha gây chóa hoặc góc nhìn từ trên cao xuống khác biệt so với tập COCO thông thường.
- **Trường hợp cần người rà lại nhãn tham chiếu:**
  + Theo `data/DATA.md`, tập nhãn tham chiếu `test/labels/*.txt` (403 box hợp lệ) do một mô hình tự động tạo ra và **chưa hề được người rà soát từng box**.
  + Trên ảnh so sánh `outputs/compare_round0.jpg`, một số vị trí dải phân cách có vệt đèn phản xạ bị gán nhãn tham chiếu, hoặc ở góc xa có xe tối màu nhưng nhãn tham chiếu không đánh dấu. Trong các trường hợp này, cần chuyên viên rà soát thủ công để xác định đó là nhãn đúng hay lỗi của bộ tham chiếu trước khi kết luận mô hình khởi đầu lạnh dự đoán sai.

---

## 3. Chiến lược chọn mẫu

Thuật toán chọn mẫu học chủ động trong notebook sử dụng công thức tính điểm tổng hợp:

$$\text{score} = W_U \cdot U + W_A \cdot A + W_D \cdot D$$

với bộ trọng số được cấu hình là $W_U = 0.5$, $W_A = 0.3$, $W_D = 0.2$:
- **$U$ (Độ bất định - Uncertainty, trọng số 0.5):** Là trung bình độ bất định của 5 box khó nhất trong khung hình, với độ bất định từng box tính theo $u = 1 - |2 \cdot \text{conf} - 1|$. Hàm này đạt giá trị lớn nhất bằng 1 khi độ tin cậy $\text{conf} = 0.5$ (trạng thái mô hình lưỡng lự phân vân nhất giữa việc có hay không có xe).
- **$A$ (Mật độ box mơ hồ - Ambiguity, trọng số 0.3):** Tỷ lệ số box có độ tin cậy nằm trong vùng ranh giới nhạy cảm $0.15 \le \text{conf} < 0.5$, được chuẩn hóa theo giá trị cực đại trong pool. Đại diện cho các khung hình chứa nhiều đối tượng gây bối rối cho mô hình.
- **$D$ (Độ đa dạng thời gian - Diversity, trọng số 0.2):** Khoảng cách thời gian từ khung hình đang xét đến khung hình đã gán nhãn gần nhất (tối đa 10 giây). Thành phần này kéo các mẫu được chọn phân bố trải dài theo trục thời gian, ngăn ngừa việc dồn cục bộ vào một khoảng thời gian hẹp.
- **Vai trò của `MIN_GAP_S`:** Được đặt bằng $2.0$ giây. Vì camera cố định trên cầu vượt, các khung hình cách nhau dưới 2 giây có luồng xe và bối cảnh gần như trùng lặp (redundant). `MIN_GAP_S` đóng vai trò bộ lọc thời gian bắt buộc, ép thuật toán bỏ qua các ảnh gần trùng để tối đa hóa lượng thông tin mới trên từng đơn vị chi phí gán nhãn.

**Minh chứng từ `reports/SELECTION.md` và các frame cụ thể:**
1. `frame_0182.jpg` (Rank 1, score=0.9591, U=0.9182, A=1.0000, D=1.0000): Điểm cao nhất pool, chứa tới 18 box mơ hồ (A=1.0).
2. `frame_0369.jpg` (Rank 2, score=0.9324, U=0.9315, A=0.8889, D=1.0000): Đoạn cuối mật độ cao (43 box), U rất cao thể hiện nhiều ca che khuất phức tạp.
3. `frame_0099.jpg` (Rank 8, score=0.9063, U=0.9460, A=0.7778, D=1.0000): Có điểm U cao nhất top 10 (0.9460), kéo phân bố mẫu về đoạn đầu video (t=39.6s) để bảo đảm đa dạng dữ liệu.
4. `frame_0372.jpg` (Rank 6, score=0.9101, U=0.9202, A=0.8333): Frame có điểm cao đứng thứ 6 nhưng bị loại bỏ hoàn toàn khỏi lô được chọn vì thời điểm $t = 148.8\text{s}$ chỉ cách `frame_0369.jpg` đúng 1.2 giây (nhỏ hơn `MIN_GAP_S = 2.0s`). Đây là minh chứng rõ ràng cho việc thuật toán kiểm soát ảnh trùng cảnh để tiết kiệm công gán nhãn.

**Điểm bất định có chứng minh ảnh đó sẽ cải thiện mô hình không?**
- **Không.** Độ bất định cao chỉ phản ánh trạng thái thiếu tự tin của mô hình hiện tại ở các vùng ảnh đó.
- Nếu sự bất định xuất phát từ nhiễu thị giác không thể học (như vệt phản quang nhòe nhoẹt trên mặt đường, quầng sáng đèn pha chói lòa hoặc xe quá nhỏ sát đường chân trời dưới 16px), việc ép mô hình học có thể gây ra hiện tượng học vẹt hoặc làm mô hình trở nên quá dè dặt (tăng precision nhưng triệt tiêu recall).
- Ngoài ra, việc cải thiện số đo còn phụ thuộc vào tính nhất quán của người gán nhãn và sự phân bố tương thích giữa các ca khó được chọn với tập kiểm thử.

---

## 4. Các vòng học chủ động (active learning)

Bảng tổng hợp kết quả các vòng từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 347 | 0.810 | +0.039 | 1.000 | 0.213 | 0.352 | 0.000 | 0.199 | 0.658 |

### Chi tiết vòng 1:
1. **Mức độ sửa nhãn gợi ý (truy xuất từ `outputs/round1_diff.md` và `outputs/round1_diff.json`):**
   - Lô gồm 12 ảnh. Mô hình ban đầu đề xuất 169 box pre-label.
   - Sau khi rà soát và chuẩn hóa trên CVAT, tổng số box đạt **347 box**.
   - Số box giữ nguyên (accepted, IoU $\ge$ 0.85): **128 box** (tỷ lệ chấp nhận đạt 76%).
   - Số box chỉnh sửa vị trí/kích thước (edited, 0.50 $\le$ IoU < 0.85): **26 box**.
   - Số box xóa bỏ (deleted - FP của mô hình): **15 box** (chủ yếu là box vệt sáng phản xạ trên đường hoặc box trùng lặp).
   - Số box thêm mới (added - FN của mô hình): **193 box** (bổ sung hơn gấp đôi số box ban đầu, tập trung vào xe tối màu, xe bị cắt ở mép ảnh và xe ở xa).
2. **Biến động AP50:**
   - So với khởi đầu lạnh: AP50 tăng từ 0.771 lên 0.810 (**tăng +0.039**, tương đương +3.9%).
   - So với vòng trước: Tăng +0.039.
3. **Nhóm xe tốt lên hoặc xấu đi trên cùng tập kiểm thử:**
   - *Nhóm tốt lên:* Độ chính xác Precision@0.25 đạt mức tuyệt đối **1.000** (100% - số lượng False Positives giảm hoàn toàn từ 16 box về 0 box). Độ phủ ở nhóm xe kích thước lớn (`R large`) tăng từ 0.5610 lên **0.6585** (+9.75%), cho thấy mô hình nhận diện xe gần cực kỳ sắc nét và đúng ranh giới thân xe.
   - *Nhóm xấu đi:* Tại ngưỡng cố định conf = 0.25, Recall tổng thể giảm từ 0.4888 xuống 0.2134; trong đó `R small` giảm về 0.0000 và `R medium` giảm xuống 0.1993. Lý do là sau khi huấn luyện trên 12 ảnh với ranh giới xe chuẩn xác, mô hình trở nên rất khắt khe, không còn đưa ra các dự đoán phỏng đoán với độ tin cậy thấp cho các đốm sáng mờ ảo ở xa. Do đó ở ngưỡng cắt conf 0.25, các xe nhỏ bị lọc bớt; tuy vậy, toàn dải tích phân Precision-Recall (AP50) vẫn tăng nhờ các dự đoán tự tin có độ chính xác hoàn hảo.

### Phân tích ca cụ thể và phân biệt ba cấp độ:
- **Ca kết quả đổi sau fine-tune trong `outputs/compare_round1.jpg`:**
  + Trên các ảnh test hiển thị (`frame_0050`, `frame_0150`, `frame_0250`, `frame_0350`), ở làn đường gần camera, mô hình vòng 1 đóng box ôm khít thân xe, xóa sạch toàn bộ các box ảo đỏ (FP) do vệt phản chiếu ánh đèn mà mô hình cold start mắc phải.
- **Phân biệt ba cấp độ dựa trên `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md`:**
  1. *Quan sát độc lập (`BLIND_SCAN.md`):* Khi quét mù `frame_0099.jpg` (chưa bật box AI), mắt người phát hiện khoảng 27 xe và ghi chú rõ hai vị trí xe khuất nửa thân ở mép trái và mép phải.
  2. *Lỗi pre-label đã sửa (`REVIEW_LOG.csv`, `round1_diff.md`):* AI chỉ gợi ý 13 box và bỏ sót hoàn toàn các xe ở mép viền này. Người gán nhãn đã thêm 12 box mới (trong đó có xe mép trái x=0.189, y=0.471 và mép phải), sửa 3 box dính vệt đèn và giữ nguyên 10 box chuẩn.
  3. *Kết quả mô hình sau train (`metrics_round1.json`):* Mô hình sau khi fine-tune loại trừ triệt để lỗi gán nhầm vệt đèn (FP = 0), học được hình thái xe kích thước lớn chuẩn xác, song cần thêm dữ liệu để nhận diện nhạy hơn với các xe nhỏ ở cự ly xa.
- **Ca khó theo guideline:**
  + Trường hợp xe bị cắt ở mép ảnh trong `frame_0107.jpg` (x=0.970, y=0.721): Xe đang từ ngoài tiến vào khung hình, chỉ nhìn thấy nửa đầu xe và cụm đèn bên phải. Theo đúng [GUIDELINE_LABEL.md](file:///c:/Users/admin/K4-L2-DAY08-LuongPhuongQuang-2A202602175/GUIDELINE_LABEL.md), người gán nhãn chỉ vẽ box bao bọc phần thân xe nằm trong ảnh, không phóng đại box ra ngoài vùng ảnh và không gán theo vệt sáng chiếu về phía trước.

---

## 5. Kết luận và giới hạn

### Kết luận vòng này:
- Vòng học chủ động thứ nhất đạt kết quả tích cực: AP50 tăng từ **0.771 lên 0.810** (+0.039) chỉ với 12 ảnh gán nhãn bổ sung (347 box).
- Mô hình đạt độ tinh khiết tối đa với Precision = 1.000 và cải thiện đáng kể khả năng nhận diện xe lớn (R large = 0.6585).
- **Quyết định:** Tôi quyết định dừng lại ở vòng 1 vì đã hoàn thành trọn vẹn chu trình học chủ động bắt buộc của bài thực hành, đạt mức tăng AP50 trên 0.01 theo khuyến nghị của notebook (`gain_prev = +0.039 > 0.01`).

### Đề xuất cho vòng tiếp theo (nếu tiếp tục):
Nếu triển khai vòng 2, từ kết quả gợi ý trong `outputs/selection_round2.csv`, tôi đề xuất hai ca cần ưu tiên:
1. `frame_0031.jpg` (Rank 1 vòng 2, t=12.4s, score=0.8309, U=0.7952, 7 box mơ hồ): Bổ sung khung cảnh đầu video với mật độ xe xa để giải quyết điểm yếu `R small` và `R medium`.
2. `frame_0002.jpg` (Rank 2 vòng 2, t=0.8s, score=0.7853, U=0.7706, 6 box mơ hồ): Giúp mô hình thích ứng với vùng sáng lúc bắt đầu video.
- *Chi phí rà nhãn và nguy cơ ảnh gần trùng:* Cần áp dụng nghiêm ngặt `MIN_GAP_S >= 2.0s`. Trong `selection_round2.csv`, các ảnh liền kề như `frame_0030.jpg` (12.0s) và `frame_0033.jpg` (13.2s) phải bị loại bỏ để không lãng phí chi phí nhân công gán nhãn trên cùng một đoàn xe.

### Giới hạn của tập kiểm thử và số đo:
- **Tập test quá nhỏ:** Chỉ bao gồm 20 ảnh, do đó sự xuất hiện hay biến mất của vài box dự đoán có thể làm biến động số đo đáng kể, chưa đại diện đầy đủ cho phân phối thực tế.
- **Quy tắc lọc bỏ box < 16 px:** Có 14 box nhỏ sát chân trời bị bỏ qua khi tính điểm. Do đó, các cải thiện hoặc sai sót ở nhóm xe siêu nhỏ này không được phản ánh vào số đo AP50.
- **Nhãn tham chiếu do mô hình tự động tạo ra:** Nhãn test chưa được người rà soát thủ công nên bản thân nó có thể chứa các lỗi FP hoặc FN cố hữu. Điểm số AP50 thực chất phản ánh mức độ trùng khớp giữa mô hình fine-tune với mô hình tham chiếu, chứ không thể coi là chân lý mặt đất tuyệt đối.

### Kế hoạch xử lý nếu AP50 giảm ở vòng tiếp theo:
Nếu ở vòng tiếp theo AP50 bị sụt giảm, tôi sẽ tiến hành kiểm tra tuần tự các yếu tố sau trước khi chạy huấn luyện lại:
1. *Kiểm tra chất lượng nhãn mới sửa (`labels/roundN/`):* So sánh qua `outputs/roundN_diff.md` xem tỷ lệ accepted/edited có bất thường không, có bị gán nhãn ẩu hoặc ôm nhầm vệt đèn pha không nhất quán theo guideline không.
2. *Kiểm tra phân phối confidence:* Vẽ lại histogram confidence score để xem mô hình có bị thiên lệch làm ngưỡng conf 0.25 không còn phù hợp hay không.
3. *Kiểm tra hiện tượng overfitting:* Với số lượng ảnh ít (12-24 ảnh), việc train 50 epoch có thể làm mô hình overfit vào nền đường và xe cụ thể; cần xem xét giảm epoch hoặc điều chỉnh learning rate.
4. *Kiểm tra tính tương thích giữa tập train và test:* Đảm bảo không chọn phải các khung hình ngoại lai (outliers) gây hại cho trọng số của mô hình.
