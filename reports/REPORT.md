# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: PHẠM THỊ OANH

Công cụ gán nhãn đã dùng: CVAT (AnyLabeling, CVAT, SAM hoặc sửa trực tiếp file nhãn)

Báo cáo được hoàn thiện dựa trên số liệu thực nghiệm. Mọi con số phải truy được
từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round*.json` hoặc
`outputs/round*_diff.md`. Không coi nhãn test do mô hình tạo là chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

Pool và test được chia theo trục thời gian để giảm việc các khung hình gần như giống nhau xuất hiện
ở cả hai tập. Vùng đệm ở giữa làm giảm rò rỉ thông tin theo thời gian: model không thể chỉ học
thuộc cảnh, góc máy hoặc chuỗi đèn xe rồi được đánh giá trên một frame gần kề. Nếu chia ngẫu nhiên,
các frame tương tự có thể rơi vào cả train/pool và test, làm AP50, precision và recall trên test
bị cao giả tạo so với khả năng tổng quát sang thời điểm khác. Cách chia theo thời gian khó hơn nhưng
phản ánh tốt hơn việc model gặp một đoạn video mới.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Cold start có precision khá cao nhưng recall chỉ 0.489: model bỏ sót nhiều xe, nhất là xe nhỏ
(R = 0.182), trong khi xe trung bình và lớn lần lượt đạt 0.547 và 0.561. Trên `compare_round0.jpg`,
các box bỏ sót tập trung ở xe xa, xe bị tối hoặc bị che; ánh đèn và xe ở xa cũng làm box khó khớp.
Tuy nhiên, trước khi kết luận model sai cần rà lại nhãn tham chiếu ở các xe rất nhỏ hoặc bị mép ảnh
cắt. Test có 403 box tham chiếu nhưng bỏ qua 14 box cao dưới 16 px, và nhãn tham chiếu được tạo bởi
model chứ chưa được người kiểm tra toàn bộ, nên một FN có thể là lỗi của nhãn tham chiếu.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

Điểm được tính theo `score = 0.5U + 0.3A + 0.2D`: `U` là bất định trung bình của năm box khó
nhất, `A` là mức độ tập trung box confidence mập mờ, còn `D` là khoảng cách thời gian tới frame đã
gán gần nhất, bị chặn ở 10 giây và chuẩn hóa. `MIN_GAP_S = 2.0` khiến cách chọn tham lam bỏ qua
frame cách frame đã chọn dưới 2 giây để tránh rà hai ảnh gần trùng; nếu chưa đủ số lượng, khoảng
cách được nới dần.

Trong `reports/SELECTION.md`, `frame_0182.jpg` (hạng 1, score 0.9591, U = 0.9182, 18/28 box
mơ hồ), `frame_0369.jpg` (hạng 2, score 0.9324, U = 0.9315, 16/43) và `frame_0331.jpg` (hạng 5,
score 0.9154, 18/47) đều được chọn vì điểm cao và có nhiều box cần người xác nhận. `frame_0372.jpg`
(hạng 6, score 0.9101, U = 0.9202, 15/42) là ứng viên thay thế nếu ảnh hạng 3 hoặc 4 gần trùng.
Điểm U cao chỉ cho biết model đang không chắc ở các box khó; nó không chứng minh ảnh đó chắc chắn
giúp model tốt hơn. Hiệu quả còn phụ thuộc lỗi thật trong ảnh, chất lượng nhãn sau CVAT và mức độ
đa dạng so với các frame đã chọn.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

Vòng 1 dùng 12 ảnh và tạo 315 box train sau khi sửa. Từ `outputs/round1_diff.md`: pre-label có
169 box, trong đó 131 được giữ nguyên, 22 được chỉnh, 16 bị xóa và 162 box được thêm mới; accept
rate là 78%. Như vậy nhãn AI ban đầu chỉ là gợi ý, còn 315 box là nhãn đã được người sửa và dùng
để fine-tune.

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 315 | 0.409 | -0.363 | 1.000 | 0.074 | 0.139 | 0.000 | 0.054 | 0.342 |

So với cold start, AP50 giảm 0.363, recall giảm từ 0.489 xuống 0.074 và F1 từ 0.640 xuống 0.139.
Recall xe nhỏ giảm từ 0.182 xuống 0; recall xe trung bình giảm từ 0.547 xuống 0.054; recall xe lớn
giảm từ 0.561 xuống 0.342. Precision tăng lên 1.000 vì model chỉ còn một số ít dự đoán đúng,
nhưng không đủ box nên đây không phải cải thiện tổng thể.

`compare_round1.jpg` cho thấy một ca xấu đi rõ ở `frame_0050.jpg`: cold start có TP 11, FP 2,
FN 7, còn vòng 1 chỉ có TP 1, FP 0, FN 17. Ở `frame_0350.jpg`, kết quả cũng giảm từ TP 9, FP 2,
FN 14 xuống TP 2, FP 0, FN 21. Đây là kết quả sau train, không phải nhãn đã sửa. Có thể kiểm tra
trước tiên việc phân phối 12 ảnh train, số lượng box thêm rất lớn so với pre-label, cấu hình class,
ngưỡng confidence và checkpoint/model export.

Quan sát độc lập trong `BLIND_SCAN.md` là frame `frame_0099.jpg`, người quan sát thấy 22 xe và
đặc biệt lưu ý xe bị cắt ở góc dưới bên phải cùng các xe rất xa gần chân cầu. `REVIEW_LOG.csv` ghi
ba ca thêm box vì thấy thân xe và ranh giới đủ rõ; đó là lỗi thiếu của pre-label đã được sửa, không
phải bằng chứng model sau fine-tune đã học đúng. Một ca khó theo guideline là xe rất xa chỉ còn
hai chấm đèn: cần phân biệt với vệt sáng, xe bị cắt hoặc vật không phải xe, và không nên suy luận
chỉ từ confidence.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

Kết quả vòng 1 kém hơn cold start trên cùng 20 ảnh test: AP50 giảm từ 0.7714 xuống 0.4087,
recall giảm mạnh và recall xe nhỏ bằng 0. Vì vậy tôi dừng ở đây để kiểm tra dữ liệu và quy trình,
không tiếp tục train thêm khi chưa hiểu nguyên nhân suy giảm.

Nếu có vòng sau, hai nhóm nên ưu tiên rà lại là (1) `frame_0326.jpg` hoặc `frame_0331.jpg`, vì số
box cuối lần lượt là 35 và 36, có nhiều box được thêm và chi phí rà cao; (2) `frame_0369.jpg`, có
38 box cuối và 25 box thêm. Cần xem contact sheet để tránh chọn hai frame gần trùng; `frame_0372.jpg`
là phương án thay thế có score cao nhưng phải cân đối thêm chi phí 42 box. Các frame có xe nhỏ,
xe bị che và xe sát mép ảnh có bất định cao nhưng cũng tốn thời gian kiểm từng box.

Kết luận bị giới hạn bởi chỉ 20 ảnh test, luật bỏ qua 14 xe rất nhỏ dưới 16 px và nhãn tham chiếu
do model tạo chưa được người rà thủ công. Vì vậy AP50 là bằng chứng định lượng trong bộ đánh giá
này, không phải chất lượng tuyệt đối của toàn bộ video; việc precision tăng sau vòng 1 còn có thể
chỉ phản ánh model dự đoán quá ít.

Nếu AP50 giảm lần nữa, tôi sẽ kiểm tra theo thứ tự: tên ảnh và nhãn YOLO, class id và tọa độ box,
phân bố box sau khi thêm 162 ca, ảnh gần trùng giữa các tập, checkpoint/export và ngưỡng confidence;
sau đó mới xem lại augmentation, số epoch và learning rate. Tôi cũng sẽ đối chiếu trực tiếp một số
ảnh trong `compare_round*.jpg` với nhãn tham chiếu trước khi quyết định train tiếp.
