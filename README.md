**Thiết kế giải pháp Prompting ứng dụng thực tế cho Kỹ thuật Prompting Nâng cao (Logic & Chuỗi)**

**1. Mục tiêu bài tập** 

Ứng dụng kiến thức Kỹ thuật Prompting Nâng cao (Logic & Chuỗi) để thiết kế một giải pháp tự động hóa công việc thực tế, rèn luyện tư duy lập luận trước khi code.

**2. Bối cảnh & vấn đề thực tế**

Cửa hàng cần tự động hóa quy trình xử lý thông tin liên quan đến Chain-of-Thought, Prompt Chaining, xử lý đứt gãy.

**3. Yêu cầu nghiệp vụ chi tiết**

Sinh viên phải xác định rõ các thông tin cần đưa vào và cấu trúc kết quả mong muốn.

**4. Ràng buộc & bẫy dữ liệu lường trước**

AI dễ bỏ sót các trường hợp dữ liệu bị trống hoặc bị sai lệch. Thiết kế prompt phải có các câu lệnh điều kiện xử lý lỗi.

**5. Yêu cầu đầu ra chi tiết**

**Bước 1: Thiết kế giải pháp (I/O)**

Input
Nhóm	Dữ liệu	Dùng ở bước
Tin nhắn khách	Văn bản tự do (có thể sai chính tả, viết tắt, thiếu thông tin)	P1
Danh mục sản phẩm	SKU, tên, size, giá, tồn kho	P2
Chính sách	Mã giảm giá + điều kiện; phí ship theo khu vực; ngưỡng freeship	P2
Giọng điệu thương hiệu	Thân thiện, xưng “shop – bạn”	P3

Output mong muốn

Output P1 :


{
  "is_order": true,
  "customer_name": null,
  "phone": "string|null",
  "address": "string|null",
  "items": [{"raw_name": "", "size": "", "qty": 0}],
  "coupon": "string|null",
  "missing_fields": []
}

Output P2 :


{
  "reasoning": ["bước 1...", "bước 2..."],
  "status": "OK | NEED_INFO | OUT_OF_STOCK | PARTIAL",
  "item_checks": [{"sku": "", "requested": "", "stock": 0, "result": ""}],
  "errors": [{"code": "", "detail": ""}],
  "pricing": {"subtotal": 0, "discount": 0, "shipping": 0, "total": 0}
}

Output P3: Tin nhắn tiếng Việt, tối đa 120 từ, gồm: xác nhận đã nhận đơn, nêu rõ vấn đề, đề xuất giải pháp, và câu hỏi cần khách trả lời.

1.4. Bẫy dữ liệu và cách xử lý 
#	Bẫy	Quy tắc xử lý trong prompt
1	Thiếu tên / SĐT / địa chỉ	Ghi null, thêm vào missing_fields, tuyệt đối không bịa
2	SĐT sai định dạng (không đủ 10 số, không bắt đầu bằng 0)	Đánh lỗi INVALID_PHONE
3	Tên sản phẩm viết sai / mơ hồ	Chỉ khớp khi chắc chắn; không chắc → hỏi lại khách
4	Số lượng mơ hồ (“vài cái”, “mấy cái”)	Không tự gán, đánh AMBIGUOUS_QTY
5	Size/mặt hàng hết hàng	Đánh OUT_OF_STOCK, gợi ý size còn hàng
6	Mã giảm giá không đủ điều kiện	Không áp dụng, giải thích lý do
7	Tin nhắn không phải đơn hàng	is_order=false, dừng chuỗi

**Bước 2: Viết prompt và chạy thử**
Prompt 1 – Trích xuất
# VAI TRÒ
Bạn là nhân viên nhập liệu cẩn thận của cửa hàng thời trang Mây Store.

# NHIỆM VỤ
Đọc tin nhắn của khách và trích xuất thông tin đặt hàng thành JSON.

# QUY TẮC
1. Chỉ lấy thông tin CÓ TRONG tin nhắn. Thông tin không có → null. KHÔNG suy đoán, KHÔNG bịa.
2. Nếu tin nhắn KHÔNG phải yêu cầu đặt hàng → trả {"is_order": false} và dừng.
3. Số lượng không rõ ràng ("vài", "mấy") → qty = null.
4. Liệt kê mọi trường bị thiếu (customer_name, phone, address, size, qty) vào "missing_fields".
5. Giữ nguyên cách khách gọi tên sản phẩm vào "raw_name".

# ĐỊNH DẠNG ĐẦU RA
Chỉ trả về JSON hợp lệ, không giải thích, không markdown:
{"is_order":bool,"customer_name":str|null,"phone":str|null,"address":str|null,
 "items":[{"raw_name":str,"size":str|null,"qty":int|null}],
 "coupon":str|null,"missing_fields":[str]}

# TIN NHẮN KHÁCH
"""
{{customer_message}}
"""
Tin nhắn khách (chứa nhiều bẫy):

“Chào shop, mình lấy 2 áo thun trắng size M với 1 quần jean ống suông size 30 nha. Giao về 12 Nguyễn Huệ Q1. SĐT 0912 345 67. Có mã SALE10 á”

(Các bẫy: áo size M hết hàng, quần size 30 hết hàng, SĐT thiếu số, không có tên người nhận.)

Kết quả Prompt 1:

{
  "is_order": true,
  "customer_name": null,
  "phone": "0912 345 67",
  "address": "12 Nguyễn Huệ Q1",
  "items": [
    {"raw_name": "áo thun trắng", "size": "M", "qty": 2},
    {"raw_name": "quần jean ống suông", "size": "30", "qty": 1}
  ],
  "coupon": "SALE10",
  "missing_fields": ["customer_name"]
}

Kết quả Prompt 2:

{
  "reasoning": [
    "Bước 1: Thiếu tên người nhận. SĐT '0912 345 67' sau khi bỏ khoảng trắng còn 9 chữ số (091234567), không đủ 10 số nên không hợp lệ. Địa chỉ có, thuộc Q1 TP.HCM nên là nội thành.",
    "Bước 2: 'áo thun trắng' khớp AT01, size M, tồn M = 0 < 2 nên hết hàng. 'quần jean ống suông' khớp QJ02, size 30, tồn 30 = 0 < 1 nên hết hàng.",
    "Bước 3: Cả hai món đều không đủ hàng nên không có món nào để tính tiền, subtotal = 0.",
    "Bước 4: subtotal = 0 nên SALE10 không đủ điều kiện (cần ≥ 500.000đ), discount = 0. Chưa có hàng để tính ship nên shipping = 0.",
    "Bước 5: Có cả lỗi thông tin liên hệ lẫn hết hàng nên status = PARTIAL."
  ],
  "status": "PARTIAL",
  "item_checks": [
    {"sku": "AT01", "requested": "M x2", "stock": 0, "result": "OUT_OF_STOCK (còn size S:5, L:10)"},
    {"sku": "QJ02", "requested": "30 x1", "stock": 0, "result": "OUT_OF_STOCK (còn size 28:3, 29:2)"}
  ],
  "errors": [
    {"code": "MISSING_NAME", "detail": "Chưa có tên người nhận"},
    {"code": "INVALID_PHONE", "detail": "SĐT chỉ có 9 chữ số"},
    {"code": "OUT_OF_STOCK", "detail": "Áo thun trắng size M hết hàng"},
    {"code": "OUT_OF_STOCK", "detail": "Quần jeans ống suông size 30 hết hàng"},
    {"code": "COUPON_NOT_APPLIED", "detail": "Chưa đủ điều kiện áp dụng SALE10 (cần đơn từ 500.000đ)"}
  ],
  "pricing": {"subtotal": 0, "discount": 0, "shipping": 0, "total": 0, "provisional": true}
}

Kết quả Prompt 3 (tin nhắn gửi khách):

Chào bạn, shop đã nhận được yêu cầu đặt hàng của bạn ạ 😊 Hiện áo thun trắng size M và quần jeans ống suông size 30 bên shop đã hết hàng. Shop có sẵn áo thun trắng size S hoặc L, và quần jeans ống suông size 28 hoặc 29, bạn muốn đổi sang size nào ạ?

Ngoài ra, bạn vui lòng cho shop xin họ tên người nhận và kiểm tra lại số điện thoại (hiện đang thiếu một số) để shop chốt đơn nhé. Mã SALE10 sẽ được áp dụng khi đơn của bạn từ 500.000đ.
