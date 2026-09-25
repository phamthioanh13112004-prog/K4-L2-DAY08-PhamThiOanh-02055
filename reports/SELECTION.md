# Lựa chọn năm frame để rà nhãn

Dựa trên 50 dòng đầu của `outputs/selection_round1.csv`, tôi ưu tiên năm frame sau nếu chỉ
có ngân sách rà năm ảnh:

| Ưu tiên | Frame | Điểm | Thời điểm (s) | Thứ tự trong CSV | Căn cứ chọn |
| ---: | --- | ---: | ---: | ---: | --- |
| 1 | `frame_0182.jpg` | 0.9591 | 72.8 | 1 | Điểm cao nhất; U = 0.9182 và có 18 box mơ hồ trên 28 box, nên có nhiều khả năng đem lại thông tin khi kiểm tra. |
| 2 | `frame_0369.jpg` | 0.9324 | 147.6 | 2 | Điểm rất cao, U = 0.9315 và có 16 box mơ hồ trên 43 box; bổ sung một cảnh nhiều đối tượng cần rà. |
| 3 | `frame_0380.jpg` | 0.9170 | 152.0 | 3 | Nằm trong nhóm điểm cao, U = 0.9340 và có 15 box mơ hồ trên 40 box; đại diện cho đoạn thời gian cuối của video. |
| 4 | `frame_0326.jpg` | 0.9155 | 130.4 | 4 | Điểm cao, U = 0.9310 và có 15 box mơ hồ trên 39 box; giúp kiểm tra cảnh nhiều box gần nhau. |
| 5 | `frame_0331.jpg` | 0.9154 | 132.4 | 5 | Điểm gần bằng hạng 4 nhưng có 47 box và 18 box mơ hồ, là ca có chi phí rà cao và khả năng phát hiện lỗi lớn. |

## Đối chiếu với lô 12 ảnh model chọn

Ba frame thuộc lô 12 ảnh model chọn là `frame_0182.jpg` (hạng 1, điểm 0.9591,
28 box/18 mơ hồ), `frame_0369.jpg` (hạng 2, điểm 0.9324, 43 box/16 mơ hồ) và
`frame_0331.jpg` (hạng 5, điểm 0.9154, 47 box/18 mơ hồ). Cả ba đều có
`selected=True` trong CSV và có mặt trong contact sheet `outputs/selection_round1.jpg`.
Các chỉ số này cho thấy chúng vừa có điểm ưu tiên cao vừa có nhiều vùng cần con người xác nhận,
phù hợp với mục tiêu active learning.

Tôi vẫn chọn các frame ở hạng 3 và 4 dù chúng nằm gần nhau về thời điểm (152.0 s và 130.4 s
không phải là hai ảnh liên tiếp, nhưng đều thuộc các đoạn có nhiều box). Khi rà thực tế cần xem
contact sheet để loại bớt ảnh gần trùng nếu hai cảnh cho cùng một thông tin; trong trường hợp đó,
frame hạng 6 `frame_0372.jpg` là ứng viên thay thế hợp lý vì điểm 0.9101, U = 0.9202 và có
15 box mơ hồ.

## Một frame điểm cao nhưng không chọn

`frame_0372.jpg` ở hạng 6 có điểm 0.9101, U = 0.9202, 42 box và 15 box mơ hồ nhưng không nằm
trong năm frame ưu tiên. Đây là lựa chọn thay thế tốt nếu ảnh hạng 3 hoặc 4 bị gần trùng với ảnh
đã rà; việc bỏ qua nó chỉ do giới hạn ngân sách, không có nghĩa nó dễ hoặc nhãn chắc chắn đúng.

## Giới hạn của phép chọn

Phép chọn này chỉ ưu tiên ảnh để con người rà nhãn dựa trên điểm bất định, số box và thứ hạng của
model. Nó không chứng minh model có độ chính xác cao, không thay thế đánh giá trên tập test và
không cho biết nhãn nào là đúng nếu chưa xem ảnh và sửa trong CVAT. Đặc biệt, `empty=False` chỉ
cho biết model đã dự đoán box, không đảm bảo mọi xe đều được phát hiện hoặc mọi box đều đúng.
