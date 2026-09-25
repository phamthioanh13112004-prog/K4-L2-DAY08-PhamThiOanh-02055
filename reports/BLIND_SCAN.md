# Quét độc lập trước khi xem pre-label

Frame: frame_0099.jpg tên một ảnh trong `to_label/round1/images/train/`

Số xe nhìn thấy bằng mắt: số xe nhìn được bằng mắt 22 xe

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:  Chỗ 1: góc dưới bên phải có một xe bị mép ảnh cắt mất, chỉ còn đuôi và đèn hậu. Chỗ 2: gần chân cầu có vài xe rất xa, chỉ còn hai chấm đèn. Vệt sáng trên mặt đường bên trái là ánh đèn, không phải xe.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
