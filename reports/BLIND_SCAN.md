# Quét độc lập trước khi xem pre-label

Frame: frame_0099.jpg

Số xe nhìn thấy bằng mắt: 21 xe

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:
1. Các xe chạy ngược chiều ở làn bên trái (đặc biệt là xe SUV/xe con màu tối ở nửa dưới khung hình): Đèn pha rọi vệt sáng rất mạnh và dài xuống mặt đường nhựa. AI rất dễ vẽ box bị phình to, ôm trùm cả vệt sáng đèn phản chiếu trên mặt đường thay vì chỉ ôm sát ranh giới thân xe theo guideline.
2. Cụm xe ở rất xa phía trước bên phải và xe bị cắt mép viền: Các xe ở xa chỉ nhìn thấy hai chấm đèn hậu màu đỏ nhỏ mờ, thân xe tối chìm vào cảnh đêm nên mô hình dễ bỏ sót (False Negative). Ngoài ra, xe ở góc dưới cùng bên phải bị cắt mép khung hình chỉ nhìn thấy một phần nóc/thân cũng là vị trí AI dễ bỏ qua.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
