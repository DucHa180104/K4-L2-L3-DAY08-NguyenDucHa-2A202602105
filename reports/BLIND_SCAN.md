# Quét độc lập trước khi xem pre-label

Frame: `frame_0187.jpg` tên một ảnh trong `to_label/round1/images/train/`

Số xe nhìn thấy bằng mắt: **26 xe**

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe: 
1. **Sát mép dưới, gần giữa ảnh:** một xe tối màu bị cắt mất phần lớn thân, chỉ còn phần trên của xe trong khung hình. Xe này dễ bị bỏ sót vì không thấy trọn xe. Khi gán nhãn, chỉ khoanh phần xe còn nhìn thấy trong ảnh, không kéo box ra ngoài mép ảnh.
2. **Nhóm xe ở xa phía trên bên trái của phần đường:** thân xe tối, các cụm đèn hậu đỏ nhỏ và nằm gần nhau. Vùng này dễ bị bỏ sót xe hoặc gộp nhầm hai xe vào một box. Cần phân biệt từng xe theo phần thân và cụm đèn; không khoanh riêng ánh sáng phản chiếu trên mặt đường. Với xe quá xa, box cao dưới khoảng 16 pixel, guideline cho phép gán hoặc bỏ qua.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
