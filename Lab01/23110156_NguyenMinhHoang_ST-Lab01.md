# Báo cáo Bài Tập Thực Hành STLab01

## Thông tin sinh viên
* **MSSV:** 23110156
* **Họ tên:** Nguyễn Minh Hoàng
* **Business Flow:** BF-07 - Checkout Online Payment

## 1. Mô tả Business Flow - Checkout Online Payment
* **Actor:** User/Payment Gateway (System or Third party)
* **Mô tả:** mỗi người dùng sau khi mua hàng sẽ đều phải đi qua bước thanh toán, thông thường những nền tảng ecommerce hay các nền tảng khác có 2 phương thức thanh toán cơ bản, đó là thanh toán COD (tiền mặt) hoặc thanh toán trực tuyến (VNPay, Momo, Online Banking....). Dưới đây sẽ là miêu tả về luồng hoạt động của cách thức thanh toán online:

    | Steps | Actor | Main Action | System State/Outcome |
    |-----|-------|-------------|-------------|
    | **1. User chọn vào đơn hàng muốn thanh toán** | User | Bấm nút thanh toán, sau đó kiểm tra thông tin đơn hàng và chọn phương thức thanh toán trực tuyến (VNPay, Momo, Online Banking,....) | Hệ thống sẽ tạo đơn hàng với trạng thái là `Pending` (chờ thanh toán). |
    | **2. Mã hóa và điều hướng** | System | Hệ thống sẽ tạo URL chứa mã đơn hàng, số tiền và khóa bảo mật (secret key hoặc signature), sau đó chuyển hướng người dùng. | Trình duyệt của người dùng sẽ chuyển sang giao diện cổng thanh toán mà người dùng chọn (Payment Gateway). |
    | **3. Thực hiện thanh toán** | User | Nhập thông tin thẻ/ví điện tử hoặc quét mã QR của cổng thanh toán. | Tiền bị trừ tài khoản người dùng. Trạng thái giao dịch trên `Gateway` là `success`. | 
    | **4. Webhook/IPN Validation** | Payment Gateway (VNPay, Momo, Bank....) | Gửi 1 request ẩn về server của hệ thống để thông báo kết quả giao dịch | Hệ thống tiếp nhận payload, kiểm tra tính hợp lệ của chữ ký (signature hoặc secret key) và số tiền. Sau đó cập nhật `OrderStatus` thành `Paid` và trả về cho `Gateway` `HttpStatusCode 200`. | 
    |**5. Điều hướng User về trang kết quả**| User | `Gateway` sau khi nhận được code 200 sẽ tự động điều hướng user về trang kết quả `Return URL` của website. | Giao diện Frontend truy vấn trạng thái đơn hàng từ Backend và hiển thị "Đặt hàng thành công". |

### 1.1 Template 1 - Requirement Analysis 
| Req ID | Requirement | Actor | Rule / Constraint | Question / Assumption | Risk |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **REQ-07** | Create payment (Tạo phiên thanh toán trực tuyến). | User | Tổng tiền giỏ hàng phải > 0 để chọn thanh toán Online. | 1. Việc chặn out-of-stock đã được làm ở bước Add to Cart hay chưa? <br>2. Hệ thống có kiểm tra lại tồn kho lần cuối trước khi tạo URL redirect sang Gateway không? Ví dụ: User A thêm vào giỏ hàng món hàng lúc 8:00 sáng nhưng tới 10:00 sáng mới thanh toán và lúc đó thì món hàng lại bị người khác mua mất và không còn trong tồn kho nữa. | Concurrency (Xung đột dữ liệu): 2 user cùng thanh toán cho món hàng cuối cùng. User bị trừ tiền ở Gateway nhưng hệ thống không có hàng để giao. |
| **REQ-08** | Redirect/callback (Chuyển hướng sang Gateway). | System | Hệ thống sinh ra URL chứa mã đơn hàng, số tiền và mã băm (secure hash) hợp lệ. | 1. Có timeout cho trang gateway không? Vì nếu User cứ treo máy và làm việc khác trong lúc thanh toán thì sao? <br>2. Nếu User bấm back về web thì có được thanh toán lại cho đơn đó không? <br>3. Nếu Api Payment Gateway sập, thì UI sẽ báo lỗi gì và sẽ hướng dẫn user xử lý cho đơn muốn thanh toán đó như thế nào? | 1. Nếu không có timeout, lỡ như User để treo trang `Gateway` đó và làm việc khác thì sản phẩm sẽ bị giữ vô thời hạn. <br>2. User đóng trình duyệt ngay khoảnh khắc đơn hàng đang được thanh toán, tiền chưa trừ nhưng đơn hàng bị kẹt ở trạng thái Pending vô thời hạn. |
| **REQ-09** | Payment Webhook (Cập nhật kết quả ngầm). | Payment Gateway | Chỉ chấp nhận request có chữ ký hay khóa bảo mật(signature/secret-key) hợp lệ và số tiền khớp đúng với Total của đơn hàng. | Gateway gửi webhook trễ (delay) hoặc gửi trùng lặp 2 lần cho cùng một giao dịch thành công thì xử lý thế nào? | User bị trừ tiền nhưng webhook lỗi/trễ khiến hệ thống không ghi nhận "Paid", user bị mất tiền oan. |
| **REQ-10** | Confirm order (Xác nhận đơn và hiển thị). | System / User | Trạng thái chuyển thành "Paid" và giỏ hàng (Cart) phải được làm trống. | 1. Giao diện Return URL có nhiệm vụ hiển thị gì để cho User có thể chắc chắn mình đã đặt hàng thành công? <br>2. Return URL phải sử dụng cơ chế hay kĩ thuật gì để ngăn chặn việc nếu User spam refresh lại trang mà không gây ra lỗi hay quá tải request cho hệ thống? | Return URL bị user cố tình F5 (refresh) nhiều lần gây lỗi hiển thị hoặc lỗi gọi API kiểm tra trạng thái thừa thãi. |

### 1.2 Template 2 - Business Flow Map 
| Flow ID | Flow Name | Actor | Precondition | Main Steps | Outcome | Risk
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **BF-07.1** | Khởi tạo và điều hướng | User | Giỏ hàng có sản phẩm, đang ở trang Checkout. | 1. User chọn thanh toán Online (VNPay/MoMo).<br>2. Bấm "Place Order".<br>3. Hệ thống tạo đơn hàng Pending và URL chứa hash bảo mật. | User được chuyển hướng (redirect) thành công sang trang của Gateway. | Lỗi mạng khiến redirect thất bại, đơn hàng đã tạo nhưng user không thể thấy link thanh toán (URL hash lỗi). |
| **BF-07.2** | Thanh Toán | User | Trình duyệt đã chuyển sang trang của Payment Gateway. | 1. User nhập thông tin thẻ/ví hoặc quét mà QR trên màn hình.<br>2. Nhập mã OTP xác thực.<br>3. Xác nhận thanh toán. | Gateway báo trừ tiền thành công và chuẩn bị redirect user về lại website (Return URL). | User nhập sai OTP quá số lần, hoặc đóng trình duyệt giữa chừng, flow rẽ nhánh sang Failed nhưng hệ thống chưa biết. |
| **BF-07.3** | Đồng bộ Webhook | Payment Gateway | Giao dịch tại Gateway đã hoàn tất (Success/Fail). | 1. Gateway gọi API Webhook ngầm về hệ thống.<br>2. Hệ thống verify chữ ký/khóa (signature/secret key) & số tiền.<br>3. Cập nhật Order = Paid. | Đơn hàng được cập nhật trạng thái chuẩn xác dưới database. Cart bị xóa. API trả về HTTP 200 cho Gateway. | Webhook bị giả mạo (tampering) payload; Gateway gửi Webhook trễ hoặc gửi lặp lại nhiều lần gây lỗi database. |
| **BF-07.4** | Xác nhận thanh toán và điều hướng user về return URL để thông báo kết quả | User | User được Gateway tự động redirect ngược về Return URL. | 1. Trình duyệt tải trang Return URL.<br>2. UI truy vấn trạng thái đơn hàng hiện tại từ backend.<br>3. Hiển thị thông báo theo trạng thái. | User nhìn thấy thông báo đặt hàng thành công và mã đơn. | User về lại Return URL quá nhanh khi Webhook chưa kịp xử lý xong, UI báo "Chờ thanh toán" làm cho user hoang mang. |

### 1.3 Template 3 - Test Scenario Matrix
| ID | Scenario | Trace | Level | Type | Approach | Priority |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **TS-07.01** | User thanh toán thành công -> Webhook xác thực đúng -> Return URL hiển thị Order Success. | REQ-08, 09, 10<br>(BF-07.2, 07.3) | System | Functional | Black-box + Dynamic | P0 |
| **TS-07.02** |  User bấm "Hủy giao dịch" tại Gateway -> Return URL báo lỗi -> Order giữ nguyên Pending/Cancelled. | REQ-08, 10<br>(BF-07.2, 07.4) | System | Functional | Black-box + Dynamic | P1 |
| **TS-07.03** | Hệ thống nhận được Webhook với Order ID đúng nhưng chữ ký (signature) bị sai. | REQ-09<br>(BF-07.3) | Integration | Security | Gray-box + Dynamic | P0 |
| **TS-07.04** | Webhook hợp lệ, nhưng số tiền thanh toán (amount) nhỏ hơn tổng tiền (total) của đơn hàng Pending. | REQ-09<br>(BF-07.3) | Integration | Functional | Gray-box + Dynamic| P0 |
| **TS-07.05** | **Idempotency (Duplicate Webhook):** Gateway gửi cùng một Webhook "Success" 2 lần liên tiếp cho cùng 1 đơn hàng. | REQ-09, 10<br>(BF-07.3) | Integration | Error Handling | Gray-box + Dynamic | P0 |
| **TS-07.06** | User load trang Return URL *trước* khi backend kịp xử lý xong Webhook payload. | REQ-09, 10<br>(BF-07.4) | System | Functional | Black-box + Dynamic | P1 |

### 1.4 Template 4 - Detailed Test Case
**Test Case 1: Happy Path (Functional)**
| Field | Student Input |
| :--- | :--- |
| **Test Case ID** | TC-07-01 |
| **Title / Objective** | Xác nhận đặt hàng thành công và chuyển trạng thái thông qua thanh toán trực tuyến. |
| **Requirement / Flow Trace** | REQ-08, REQ-09, REQ-10 / BF-07.2, BF-07.3, BF-07.4 |
| **Classification** | Level: System \| Type: Functional \| Approach: Black-box + Dynamic|
| **Priority / Risk** | P0 / Critical business flow. |
| **Preconditions** | User đã đăng nhập. Giỏ hàng đang có 2 sản phẩm (Total: 200,000đ). Môi trường đã kết nối với Sandbox Payment Gateway (VNPay/MoMo). |
| **Test Data** | Tài khoản thẻ test hợp lệ do Gateway cấp (vd: thẻ ngân hàng nội địa test của VNPay). |
| **Steps** | 1. Ở trang Checkout, chọn phương thức "Thanh toán trực tuyến" và bấm "Place Order".<br>2. Tại giao diện Gateway, nhập thông tin thẻ test và mã OTP.<br>3. Bấm xác nhận thanh toán và chờ Gateway tự động redirect về Return URL của website. |
| **Expected Result** | 1. Trình duyệt hiển thị trang "Đặt hàng thành công".<br>2. Dưới database, trạng thái của Order chuyển từ "Pending" sang "Paid".<br>3. Giỏ hàng của user đã được làm trống. |
| **Actual Result** | Trình duyệt hiển thị thành công, DB đã update trạng thái "Paid", giỏ hàng trống. |
| **Status** | Pass |
| **Evidence** | Giao diện đặt hàng thành công + order status (PAID) |
| **Notes / Defect ID** | N/A |

---

**Test Case 2: Risk-based / Negative (Idempotency)**
| Field | Student Input |
| :--- | :--- |
| **Test Case ID** | TC-07-05 |
| **Title / Objective** | Xác thực idempotency của hệ thống khi nhận được duplicate Webhook từ Gateway. |
| **Requirement / Flow Trace** | REQ-09, REQ-10 / BF-07.3 |
| **Classification** | Level: Integration \| Type: Error Handling \| Approach: Gray-box |
| **Priority / Risk** | P0 / Rủi ro crash database hoặc gửi email xác nhận 2 lần cho khách hàng. |
| **Preconditions** | Đơn hàng #1001 đang ở trạng thái "Pending". Đã chuẩn bị sẵn API Testing Tool (Postman). |
| **Test Data** | Một payload Webhook hợp lệ giả lập từ Gateway báo thanh toán thành công cho Order #1001 (đúng signature, đúng amount). |
| **Steps** | 1. Dùng Postman gửi payload Webhook tới API `/api/payment/webhook`.<br>2. Kiểm tra database xem Order #1001 đã chuyển sang "Paid" chưa.<br>3. Gửi lại **chính xác payload đó** một lần nữa qua Postman (giả lập Gateway bị delay/retry). |
| **Expected Result** | Lần 1: API trả về HTTP 200 OK, update Order thành "Paid".<br>Lần 2: API vẫn trả về HTTP 200 OK để xác nhận với Gateway, nhưng **không** update lại DB, **không** insert thêm dòng mới vào bảng `transactions`, và hệ thống không sập. |
| **Actual Result** | Lần 2 API trả về lỗi `HTTP 500 Internal Server Error` do lỗi `UniqueConstraintViolation` ở bảng transaction log. |
| **Status** | Fail |
| **Evidence** | `TC0705_postman_500_response.json` + `TC0705_backend_error.log` |
| **Notes / Defect ID** | DEF-07-01 |

### 1.5 Template 5 - Exploratory Testing Charter

| Charter Field | Student Input |
| :--- | :--- |
| **Mission** | Khám phá các rủi ro (edge cases) liên quan đến Race Condition (xung đột thời gian) giữa Return URL (UI) và Webhook (Backend API). |
| **Scope** | Checkout Page, Return URL UI, Webhook API endpoint. |
| **Timebox** | 20–30 phút |
| **Test Data / Environment** | Môi trường Staging, kết nối Sandbox Gateway. Sử dụng Chrome DevTools để throttle mạng xuống 'Slow 3G'. |
| **Risks / Ideas to Explore** | Điều gì xảy ra nếu mạng lag, trình duyệt của user load trang Return URL *trước khi* Gateway kịp gửi Webhook về Backend? UI sẽ báo lỗi, báo thành công, hay sập? Điều gì xảy ra nếu user bấm F5 (Refresh) liên tục ở trang Return URL? |
| **Observations** | Khi bóp băng thông mạng (Slow 3G), user bị redirect về Return URL trước khi Webhook chạy xong. Do database vẫn là "Pending", UI lập tức hiển thị lỗi: "Thanh toán chưa hoàn tất / Chờ thanh toán", dù user đã bị trừ tiền. 5 giây sau user bấm F5 thì UI mới đổi thành "Thanh toán thành công". Điều này gây hoang mang cho UX. |
| **Defects / Questions** | UI ở trang Return URL có nên áp dụng cơ chế Polling (gọi API check liên tục mỗi 2 giây) và hiển thị màn hình chờ "Đang xác thực thanh toán..." thay vì lập tức chốt kết quả hiển thị không? |
| **New Regression Ideas** | Cần bổ sung thêm kịch bản Automation UI: Load trang Return URL -> Verify hiển thị skeleton loading -> Bắn webhook API ngầm -> Verify UI tự động update thành "Success". |
| **Evidence** | `network_delay.png` + `ui_premature_unpaid_state.mp4` |

---

### 1.6 Template 6 - Defect Report

| ID | Title | Severity | Precondition | Steps | Expected | Actual | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **DEF-07-01** | Duplicate webhook kích hoạt HTTP 500 error (Thiếu Idempotency). | High | Đơn hàng #1001 đang ở trạng thái "Pending". Đã chuẩn bị sẵn payload Webhook "Success" hợp lệ trên Postman. | 1. Bắn payload webhook lần 1 qua Postman.<br>2. Gửi chính xác payload webhook đó thêm lần 2. | Lần 2 hệ thống phải nhận diện được giao dịch đã xử lý, giữ nguyên trạng thái "Paid" và trả về HTTP 200 OK cho Gateway. | Lần 2 hệ thống crash và trả về HTTP 500 do lỗi `UniqueConstraintViolation` (Cố gắng insert trùng transaction ID vào DB). Gateway sẽ tưởng lỗi và tiếp tục gửi lại liên tục. | `postman_logs.txt`, DB error log: `Duplicate entry for key...` |

### 1.7 Template 7 - Test Execution Summary

| Summary Field | Student Input |
| :--- | :--- |
| **Scope Tested** | BF-07 Checkout – Online Payment. |
| **Build / Environment** | Môi trường Staging (Kết nối Sandbox Payment Gateway). |
| **Total / Passed / Failed / Blocked** | Total: 6 \| Passed: 4 \| Failed: 1 \| Blocked: 1 |
| **Critical / High Defects** | DEF-07-01: Thiếu cơ chế Idempotency gây crash HTTP 500 khi nhận duplicate Webhook. |
| **Coverage Summary** | 100% Main flow & Alternative flow đã được test. Các rủi ro về Race Condition và giả mạo bảo mật (tampering) đã được cover cơ bản. |
| **Not Tested / Remaining Risk** | Chưa test behavior của hệ thống khi có lượng lớn giao dịch đồng thời. <br>Chưa test các case liên quan đến sai lệch tỷ giá tiền tệ (nếu Gateway là quốc tế). |
| **Next Action** | 1. Dev team cần fix gấp DEF-07-01 (Idempotency).<br>2. BA/PO cần trả lời câu hỏi về cơ chế Polling cho Return URL ở Charter 5. |

## 2. Hai bài toán mới
### 2.1 MediBook: Đặt lịch khám trực tuyến
MediBook cho phép bệnh nhân tìm bác sĩ, chọn khung giờ, đặt lịch và nhận xác nhận. Bác sĩ/quầy tiếp nhận có thể cập nhật trạng thái lịch hẹn.
* **Requirement Analysis:**

| Req ID | Actor | Input / Test Data | Business Rule | State | Dependency | Failure Mode | Evidence | Ambiguity |
|---|---|---|---|---|---|---|---|---|
| MB-01 | Patient | Chuyên khoa, ngày khám | Chỉ hiện bác sĩ/slot còn khả dụng | Slot: Available / Booked / Blocked | Lịch làm việc bác sĩ, dữ liệu slot | 1. Slot đã đầy nhưng vẫn hiện <br>2. Bác sĩ nghỉ nhưng vẫn hiện <br>3. Ngày ở quá khứ những vẫn book được | Kết quả search vs dữ liệu slot trong DB |  |
| MB-02 | Patient | Slot, họ tên, SĐT, email | 1 slot 1 người thì không được trùng | Slot: Available → Booked | Slot service, validation | 1. Nếu như hai bệnh nhân đặt cùng slot <br>2. Dữ liệu liên hệ sai định dạng | Response API, bản ghi appointment, trạng thái slot | Slot nhận nhiều người thì quy định thế nào? Có bắt buộc đăng nhập rồi mới đặt lịch khám hay không? |
| MB-03 | System | Booking request | Appointment ID duy nhất; không duplicate khi double-click/retry | Appointment: Scheduled | Idempotency, ID generator | 1. Hai Appointment ID cho một lần đặt <br>2. retry sau timeout tạo bản ghi thứ hai | Số bản ghi trong DB, log request, ID | |
| MB-04 | Patient | Appointment ID, thời điểm hủy | Hủy trước giờ khám | Scheduled → Cancelled | Giờ hệ thống, slot service | 1. Hủy sát giờ vẫn được cho phép <br>2. slot không được trả lại <br>3. hủy lịch của người khác | Trạng thái appointment + slot sau hủy | **Chưa có giới hạn thời gian hủy** và hủy xong thì slot có mở lại không? |
| MB-05 | Receptionist/Doctor | Appointment, trạng thái mới | 1. Scheduled → Checked-in → Completed hoặc Cancelled <br>2. không nhảy trạng thái trái quy trình nếu không có quyền | Scheduled, Checked-in, Completed, Cancelled | Phân quyền (RBAC) | Nhảy Scheduled → Completed; Patient tự đổi trạng thái; sửa trạng thái cuối | Lịch sử đổi trạng thái (audit log), response lỗi 403 | 1. Ai được phép nhảy trạng thái? <br> 2. Completed có quay lại được không? |
| MB-06 | System | Cấu hình thông báo | Gửi xác nhận/reminder theo cấu hình | Notification: Pending/Sent/Failed | Email/SMS service | 1. Gửi trùng, không gửi, gửi sai giờ <br>2. Lỗi gửi làm hỏng booking | Notification log, trạng thái appointment | Notification fail có rollback appointment không? Retry bao nhiêu lần? |
* **6 Requirements cần BA/PO làm rõ:**
    1. **MB-04:** Bệnh nhân được hủy trước giờ khám tối đa bao lâu (vd: trước 2h/24h)? Hủy quá hạn thì chặn hay vẫn cho hủy nhưng ghi nhạn?
    2. **MB-04:** Khi hủy thành công, slot có tự động mở lại cho người khác đặt không?
    3. **MB-06:** Notification gửi lỗi có làm booking thất bại không, hay booking vẫn thành công và gửi lại sau? Retry mấy lần?
    4. **MB-02:** Có trường hợp slot nhận được nhiều người không? Nếu có, sức chứa tối đa của slot đó là bao nhiêu?
    5. **MB-03:** Thế nào là "duplicate"? (cùng bệnh nhân + cùng slot? cùng SĐT? cùng request?) Hệ thống nhận diện như thế nào để ngăn chặn?
    6. **MB-01/02:** Bệnh nhân có cần đăng nhập không? Một bệnh nhân được đặt tối đa bao nhiêu lịch cùng ngày/cùng bác sĩ?
* **Business Flows**

| Flow | Actor | Main Path | Key Risks | Requirement Trace |
|---|---|---|---|---|
| BF-A1 Tìm bác sĩ & slot | Patient | Chọn chuyên khoa + ngày → hệ thống lọc → hiển thị bác sĩ/slot trống | Hiện slot đã đầy/đã qua hay bác sĩ nghỉ vẫn hiện | MB-01 |
| BF-A2 Đặt lịch | Patient | Chọn slot → nhập thông tin → submit → Appointment ID | 1. Hai người cùng slot -> thiếu validation -> slot không khóa | MB-02, MB-03 |
| BF-A3 Chống trùng khi double-click/retry | Patient/System | Submit lần 1 → mạng chậm → submit lần 2/retry → chỉ 1 appointment | Duplicate appointment, 2 ID cho 1 lần đặt | MB-03 |
| BF-A4 Hủy lịch | Patient | Mở lịch hẹn → Hủy → xác nhận → Cancelled → trả slot | Hủy sát giờ; slot không trả lại; hủy lịch người khác | MB-04 |
| BF-A5 Cập nhật trạng thái | Receptionist/Doctor | Scheduled → Checked-in → Completed/Cancelled | Nhảy trạng thái trái quy trình hay sai quyền | MB-05 |
| BF-A6 Thông báo | System | Booking thành công → gửi xác nhận → reminder theo cấu hình | 1. Gửi trùng/không gửi <br>2. Fail ảnh hưởng booking | MB-06, MB-03 |

* **Risk Map**

| Risk | Failure | Priority | Why it matters | Likely test focus |
|---|---|---|---|---|
| R1 — Đặt trùng slot (BF-A2, MB-02) | Hai bệnh nhân cùng đặt thành công một slot chỉ nhận 1 người (race condition, slot không được khóa kịp) | Critical | Xung đột lịch khám ngoài đời thực: hai người đến cùng giờ, bác sĩ quá tải, bệnh nhân phải chờ hoặc ra về | Hai request đồng thời cho cùng slot; kiểm tra số appointment/slot trong DB, response của request thứ hai, trạng thái slot sau đặt |
| R2 — Duplicate appointment (BF-A3, MB-03) | Double-click hoặc retry sau timeout tạo hai Appointment ID cho một lần đặt | High | Bản ghi trùng, chiếm nhiều slot, gửi thông báo trùng, khó đối soát và hủy | Double-click, retry khi mạng chậm, gửi lại cùng request; đếm bản ghi, kiểm tra idempotency và trạng thái nút submit |
| R3 — Sai quy trình/quyền đổi trạng thái (BF-A5, MB-05) | Nhảy trạng thái trái quy trình (Scheduled → Completed) hoặc người không có quyền (Patient) đổi trạng thái | High | Sai hồ sơ khám và dữ liệu vận hành; khó truy vết nếu không có audit log | Ma trận chuyển trạng thái hợp lệ/không hợp lệ theo từng vai trò; kiểm tra mã lỗi 403, audit log, trạng thái cuối |
| R4 — Hủy lịch sai quy tắc (BF-A4, MB-04) | Hủy sát giờ vẫn được, hủy lịch của người khác, hoặc hủy xong slot không được trả lại | High | Mất doanh thu và slot trống; lỗi quyền hủy lịch người khác gây lộ/ảnh hưởng dữ liệu; requirement thiếu giới hạn thời gian nên dễ hiểu sai | Hủy ở các mốc thời gian (biên); hủy bằng ID của người khác; kiểm tra trạng thái slot sau hủy (cần BA xác nhận rule) |
| R5 — Notification ảnh hưởng booking (BF-A6, MB-06) | Gửi lỗi làm booking fail/rollback sai, hoặc gửi trùng/không gửi/sai giờ | Meidum | Bệnh nhân không biết lịch hẹn hoặc nhận thông báo sai; có thể bỏ lỡ lịch khám; chưa rõ rule nên dễ tranh cãi | Giả lập email/SMS service lỗi; kiểm tra appointment vẫn đúng trạng thái, notification log, retry, không gửi trùng |
| R6 — Hiển thị slot không khả dụng (BF-A1, MB-01) | Tìm kiếm hiện slot đã đầy, đã qua giờ hoặc bác sĩ nghỉ | Medium | Bệnh nhân chọn slot không đặt được, trải nghiệm kém; dữ liệu cũ còn dẫn tới đặt nhầm | Tìm theo chuyên khoa/ngày; so khớp kết quả với dữ liệu slot; test slot vừa đầy/vừa hủy (cache), ngày quá khứ |

* **Chọn BF-A2 (Đặt lịch) cùng R1 — hai bệnh nhân đặt cùng slot.**

    - Đây là rule cốt lõi của hệ thống: sai thì hậu quả trực tiếp ở thực tế (hai bệnh nhân đến cùng giờ, bác sĩ quá tải, bệnh nhân phải chờ/về).
    - Khó phát hiện bằng test thông thường vì chỉ xảy ra khi có **conflict**.
    - Là điểm giao của nhiều flow: BF-A1 (hiển thị slot), BF-A3 (idempotency), BF-A4 (trả slot), BF-A6 (thông báo) đều phụ thuộc trạng thái slot đúng.
    - Priority cao vì nhiều bệnh nhân thường tranh các slot thuận tiện (giờ vàng, bác sĩ giỏi hay nổi tiếng, uy tín).
    Cần quan sát: số bản ghi appointment cho cùng slot trong DB, trạng thái slot sau mỗi request, response của request thứ hai ( bị từ chối rõ ràng), log theo timestamp.
 


### 2.2. Bài toán B — SmartCampus Library

* Requirement Analysis
 
| Req ID | Actor | Input / Test Data | Business Rule | State | Dependency | Failure Mode | Evidence | Ambiguity |
|---|---|---|---|---|---|---|---|---|
| LB-01 | Student | Tên/tác giả/chủ đề | Sách không còn bản vẫn xem được nhưng không mượn ngay | Copy: Available / On loan / Reserved  (dành cho số sách không còn bản nên sẽ được đặt trước) / Lost / Damaged / Restock | Catalog, số lượng bản khả dụng | 1. Số bản khả dụng sai <br>2. Cho mượn khi hết bản | Kết quả search vs số copy Available trong DB | 1. Khi tìm theo thông tin, thì có phân biệt hoa/thường hay "dấu Tiếng Việt" không? <br>2. Những sách có trạng thái "Reserved/Lost/Damaged/Restock" thì có được xem là "khả dụng" hay không? |
| LB-02 | Student | Đầu sách hết bản | Mỗi user chỉ có 1 reservation active cho cùng đầu sách | Reservation: Active / Fulfilled / Cancelled / Expired | Hàng đợi reservation | 1. Hai reservation cho cùng đầu sách <br>2. Sai thứ tự hàng đợi dẫn đến tranh chấp về reservation | Bản ghi reservation, vị trí hàng đợi | 1. Reservation giữ bao lâu? <br>2. Còn bản thì có được giữ chỗ không? <br>3. Giới hạn tổng số reservation là bao nhiêu? |
| LB-03 | Librarian/System | Student ID, Copy ID | Một copy chỉ cho một user mượn tại một thời điểm | 1. Copy: Available → On loan <br>2. Loan: Active | Quyền thủ thư, hạn mức mượn | 1. Hai Loan cùng thời điểm cho cùng copy <br>2. Mượn quá hạn mức <br>3. Mượn khi đang nợ phí | Bản ghi Loan, trạng thái Copy | 1. Hạn mức số sách được mượn là bao nhiêu? <br>2. Thời hạn mượn là bao lâu? <br>3. Có chặn khi đang có phí chưa trả? |
| LB-04 | Student/Librarian | Loan ID/Copy ID | Trả sách thì cập nhật copy về Available | 1. Loan: Active -> Returned<br>2. Copy: On Loan -> Available/Reserved (vì có thể sẽ có người đặt sách đó) | Reservation queue | 1. Copy không đổi trạng thái <br>2. Reservation kế tiếp bị bỏ qua | Trạng thái Copy | Có reservation kế tiếp thì copy Reserved hay Available? |
| LB-05 | System | Ngày hạn, ngày trả | Phí theo số ngày quá hạn | Loan: Overdue | Lịch ngày lễ, cấu hình phí | Sai số ngày, sai làm tròn, tính phí lễ | Số ngày, mức phí, bản ghi phí | Chưa có mức phí/ngày, ngày lễ, làm tròn |
| LB-06 | Student | Tài khoản đăng nhập | Chỉ xem dữ liệu của chính mình | Session/Auth | Auth, phân quyền | Xem được dữ liệu người khác | Response API với ID của user | Xem được lịch sử cũ trong bao lâu? Thủ thư xem được của ai? |
 
* **6 Requirements cần BA/PO làm rõ:**
 
1. **LB-05:** Mức phí trễ hạn là bao nhiêu/ngày, làm tròn theo ngày/giờ thế nào (ví dụ trễ 1 giờ chưa tới 1 ngày thì tính như thế nào)?
2. **LB-05:** Ngày lễ, cuối tuần, ngày thư viện đóng cửa có được tính vào số ngày quá hạn không?
3. **LB-04:** Khi trả sách có reservation kế tiếp, copy chuyển sang Reserved (giữ cho người đặt) hay Available? 
4. **LB-02:** Reservation active hết hạn khi nào, và khi sách có lại mà người đặt không đến nhận thì chuyển cho ai?
5. **LB-03:** Hạn mức số sách mượn cùng lúc và thời hạn mượn là bao nhiêu? Sinh viên đang có phí chưa thanh toán có bị chặn mượn không?
6. **LB-06:** Thủ thư/admin có được xem dữ liệu mượn của sinh viên khác không? Lịch sử lưu bao lâu?

* **Business Flows**
 
| Flow | Actor | Main Path | Key Risks | Trace |
|---|---|---|---|---|
| BF-B1 Tìm sách & xem tồn | Student | Nhập từ khóa(tên/tác giả/chủ đề) → xem danh sách + số bản khả dụng | Số bản sai; cho mượn khi hết | LB-01 |
| BF-B2 Giữ chỗ | Student | Sách hết bản → Reserve → vào hàng đợi | Reservation trùng hay sai thứ tự hàng đợi | LB-02 |
| BF-B3 Mượn sách | Librarian/System | Chọn student + copy → kiểm tra điều kiện(xem student có nợ phí hay gì không, hay copy này hiện đang có ai đang mượn không?) → tạo Loan ID → copy status: On loan | **Duplicate loan**; copy sai trạng thái | LB-03, LB-01 |
| BF-B4 Trả sách | Student/Librarian | Quét/chọn Loan → Return → copy về Available/Reserved | Reservation kế tiếp xử lý sai | LB-04, LB-02 |
| BF-B5 Tính phí trễ hạn | System | Trả muộn → tính số ngày trễ → tạo phí | Sai số ngày/làm tròn/ngày lễ | LB-05, LB-04 |
| BF-B6 Xem sách đang mượn & lịch sử | Student | Đăng nhập → xem danh sách mượn, hạn trả, lịch sử | Lộ dữ liệu người khác | LB-06 |
 
* **Risk Map**
 
| Risk | Failure | Priority | Why it matters | Likely test focus |
|---|---|---|---|---|
| R1 — Duplicate loan | Hai user cùng mượn thành công một copy tại cùng thời điểm (hai thủ thư thao tác đồng thời, double-click/retry tạo hai Loan ID) | Critical | Một cuốn sách vật lý chỉ giao được cho một người; sai thì tranh chấp, mất sách, dữ liệu mượn/trả không đối soát được | Hai request mượn đồng thời cho cùng Copy ID; double-click/retry; kiểm tra số Loan Active trên copy, trạng thái copy, response của request thứ hai |
| R2 — Availability state sai | Copy vẫn Available khi đã On loan (hoặc ngược lại); số bản khả dụng hiển thị lệch; trả sách xong copy không đổi trạng thái | Critical | Là nền cho tìm kiếm, mượn và giữ chỗ; sai ở đây kéo theo lỗi ở nhiều flow, sinh viên thấy "còn sách" nhưng không mượn được | Đối chiếu số bản khả dụng trên UI với DB sau mỗi thao tác mượn/trả/hủy giữ chỗ; test copy Lost/Damaged/Reserved có được tính vào "khả dụng" không |
| R3 — Reservation state sai | Trả sách nhưng copy bị mượn chen ngang người đặt trước; copy chuyển sai Reserved/Available; reservation không hết hạn hoặc không chuyển cho người kế tiếp | High | Mất công bằng hàng đợi, khiếu nại của sinh viên; requirement LB-04 chưa rõ copy chuyển Reserved hay Available nên dễ cài đặt và test lệch nhau | Trả sách khi có/không có reservation kế tiếp; hai người cùng chờ; thứ tự hàng đợi; reservation hết hạn |
| R4 — Tính phí trễ hạn sai | Sai số ngày quá hạn, sai làm tròn, tính phí vào ngày lễ/ngày đóng cửa | Medium | Liên quan tiền và khiếu nại; requirement chưa nêu mức phí/ngày, ngày lễ, cách làm tròn nên không được tự suy đoán expected | Trả đúng hạn, trễ 1 ngày, trễ qua ngày lễ/cuối tuần, trễ vài giờ; kiểm tra bản ghi phí |
| R5 — Access control | Student xem được danh sách mượn/lịch sử của người khác | High | Lộ dữ liệu cá nhân của sinh viên; vi phạm riêng tư dù khả năng xảy ra thấp hơn các risk còn lại | Đăng nhập tài khoản A rồi gọi API/URL với ID của B; kiểm tra mã lỗi 403, không lộ dữ liệu trong response; kiểm tra thủ thư/admin có quyền xem theo rule được xác nhận |
| R6 — Reservation trùng | Một user có hơn một reservation active cho cùng đầu sách (double-click, hai thiết bị) | High | Chiếm chỗ trong hàng đợi, ảnh hưởng người khác; vi phạm rule rõ ràng của LB-02 | Giữ chỗ hai lần liên tiếp/đồng thời; giữ chỗ khi đã có reservation active; kiểm tra số reservation active của user cho đầu sách đó |
 
* **Chọn BF-B3-Mượn sách để mô tả chi tiết:** 
 
    - **Precondition:**
    - Thủ thư đã đăng nhập, có quyền cho mượn.
    - Student có tài khoản hợp lệ, chưa vượt hạn mức, không bị chặn.
    - Copy tồn tại và đang ở trạng thái Available; không có reservation của người khác đang giữ copy này.
    - **Trigger:** Thủ thư chọn student + copy và bấm "Cho mượn".
    - **Main Steps:**
    1. Hệ thống kiểm tra quyền thủ thư và tính hợp lệ của student.
    2. Hệ thống kiểm tra trạng thái copy còn Available tại thời điểm xử lý (không dùng dữ liệu cũ trên màn hình).
    3. Hệ thống khóa, sau đó cập nhập: copy → On loan.
    4. Tạo Loan ID duy nhất, ghi ngày mượn, hạn trả.
    5. Hiển thị xác nhận kèm Loan ID, hạn trả.
    - **Outcome:** Có đúng một Loan Active gắn với copy; copy On loan; số bản khả dụng của đầu sách giảm 1; lịch sử mượn của student có bản ghi mới.
    - **Alternative/Exception:**
    - Copy vừa bị người khác mượn (hai thủ thư thao tác đồng thời) → từ chối, báo lỗi rõ, không tạo Loan.
    - Student vượt hạn mức/có phí chưa trả → từ chối hoặc cảnh báo.
    - Copy đang Reserved cho người khác → từ chối.
    - Double-click/retry sau timeout → chỉ một Loan.
    - Lỗi hệ thống giữa chừng → không để copy On loan.

---
 
# PHẦN 3 — PROBLEM SOLVING: QUICKCART
 
## 3.1. Threshold 500.000đ bị thay đổi cách tính
 
**Vấn đề:** Hai rule dùng hai mốc tính khác nhau: NEW100 xét trên **subtotal trước voucher**, free shipping xét trên **total sau voucher**.
 
**Expected behavior (cart 520.000đ):**
 
| Bước | Giá trị | Kết quả |
|---|---|---|
| Điều kiện NEW100 | Subtotal 520.000đ ≥ 500.000đ | Đủ điều kiện, áp dụng voucher |
| Sau voucher | 520.000 − 100.000 = 420.000đ | Dùng để xét free shipping |
| Free shipping | 420.000đ < 500.000đ | **Không** miễn phí ship, phải cộng phí ship |
 
**Ít nhất 3 scenario cần bổ sung/sửa:**
1. Subtotal 520.000đ + NEW100 → voucher áp dụng, total 420.000đ, **có** phí ship.
2. Subtotal 500.000đ (biên) + NEW100 → kiểm tra theo ≥ hay > cần xác nhận.
3. Subtotal 499.999đ + NEW100 → voucher bị từ chối.
4. Subtotal 600.000đ + NEW100 → total 500.000đ → biên free shipping (≥ 500.000 thì miễn phí ship hay không?).
5. Subtotal 599.999đ + NEW100 → total 499.999đ → có phí ship.

(Giả định NEW100 giảm 100.000đ theo mô tả 3.2; mức giảm và điều kiện ≥ hay > cần BA xác nhận.)
 
**Requirement**: tách thành hai rule với hai base khác nhau, Flow BF-04 (áp voucher) và flow tính phí ship/checkout.

**Risk nếu trước đó dùng cùng một threshold cho hai rule:**
 
"Dùng cùng một threshold" nghĩa là team coi 500.000đ là **một mốc chung**: cùng xét trên một con số (ví dụ subtotal) cho cả voucher và free shipping. Sau khi BA xác nhận hai rule xét trên hai giá trị khác nhau (voucher: subtotal trước voucher; free ship: total sau voucher), cách hiểu chung đó sẽ gây ra các rủi ro sau:
 
| # | Rủi ro | Ví dụ cụ thể (cart 550.000đ + NEW100, nội thành) | Hậu quả |
|---|---|---|---|
| 1 | **Test case cũ có expected sai** | Test cũ viết "550.000đ ≥ 500.000đ nên được free ship". Theo rule mới, total sau voucher chỉ 450.000đ nên **phải trả phí ship**. | Hệ thống đúng lại bị báo **fail giả**; hoặc hệ thống sai (đang free ship nhầm) lại **pass giả**, lỗi lọt qua test |
| 2 | **Dev cài đặt sai do dùng chung một biến/điều kiện** | Code dùng một điều kiện `>= 500.000` với cùng một giá trị cho cả hai rule: nếu lấy subtotal thì free ship nhầm cho cart 550.000đ; nếu lấy total sau voucher thì voucher bị từ chối nhầm vì total sau giảm thấp hơn 500.000đ. | Thất thoát doanh thu (free ship sai) hoặc khách bị tính phí/bị từ chối voucher sai, dẫn đến khiếu nại và bỏ giỏ |
| 3 | **Thiếu coverage ở điểm biên** | Team chỉ test quanh 500.000đ. Điểm biên free ship khi có voucher nằm ở **subtotal 600.000đ** (600.000 − 100.000 = 500.000) và 599.999đ nên bị bỏ sót. | Lỗi ở biên 600.000đ không được phát hiện trước khi release |
| 4 | **Traceability lẫn lộn** | Một scenario "cart 550.000đ" được trace về cả UR-04 lẫn UR-05 mà không ghi rõ phần nào kiểm rule nào. | Khi BA đổi một rule, khó biết scenario/test case nào phải sửa; dễ sửa thiếu hoặc sửa thừa |
 
**Cách ngăn chặn:** tách thành hai rule riêng, ghi rõ base tính của từng rule (UR-04: subtotal trước voucher; UR-05: total sau voucher); rà lại test case cũ để sửa expected; thiết kế test data riêng cho từng mốc biên; mỗi scenario trace rõ về một rule chính.
 
**Evidence cần quan sát:** để kết luận hệ thống tính đúng hay sai, tester cần xem các dữ liệu sau:
- **Dòng discount:** có đúng 100.000đ không, voucher có được áp dụng hay bị từ chối.
- **Dòng shipping fee:** có miễn phí hay có tính phí, và mức phí là bao nhiêu.
- **Vùng giao hàng:** nội thành hay ngoại thành, vì UR-05 chỉ cho free ship nội thành.
- **Total trên UI:** tổng cuối có khớp với phép tính (subtotal − discount + phí ship) không.
- **Response API và order total trong DB:** để chắc giá trị backend khớp với những gì UI hiển thị.

 
## 3.2. User double-click Apply Voucher
 
**Đây là defect hay UI delay?** 

Chưa thể kết luận ngay. Việc hiển thị 200.000đ (gấp đôi) dù chỉ vài giây **không phải delay thuần túy** — nó cho thấy một trạng thái sai (discount bị cộng dồn) từng được hiển thị. Nhiều khả năng là **defect** (thiếu cơ chế chống double-submit hoặc cập nhật UI sai). Mức độ phụ thuộc vào việc backend và order total cuối có đúng không.
 
**Evidence cần để kết luận:**
- Network log: có bao nhiêu request apply gửi đi, response từng cái (thứ tự, thời gian, giá trị discount).
- Backend: voucher đã bị ghi nhận dùng mấy lần (usage count, bảng voucher usage).
- Order/Cart total trong DB sau khi refresh; nếu bấm Place Order trong lúc UI hiển thị 200.000đ thì order total là bao nhiêu.
- Video/screen recording có timestamp, throttling mạng dùng để tái hiện.

**State cần kiểm tra:**
- UI: nút Apply có bị disable sau lần bấm đầu? Discount hiển thị theo từng response hay tổng cộng dồn?
- Backend: voucher gắn vào cart 1 lần hay 2 lần; xử lý idempotent không?
- Order total: discount cuối = 100.000đ, total tính đúng, voucher usage = 1.

**Phân loại scenario:**
- **Level:** System (UI + API + backend cùng tham gia; nếu kiểm tra API riêng thì Integration).
- **Type:** Functional (đúng giá trị discount) + yếu tố Non-functional nhẹ (hành vi dưới mạng chậm/concurrency).
- **Approach:** Black-box (thao tác từ UI) kết hợp Gray-box (xem log/DB); dạng kịch bản negative/exploratory với mạng bị throttle.

**Defect report cần dữ liệu để reproduce:** tài khoản và trạng thái cart (items, subtotal), voucher NEW100, điều kiện mạng (throttle/độ trễ cụ thể), thiết bị/trình duyệt/phiên bản build, các bước (bấm Apply 2 lần liên tiếp trong X ms), kết quả mong đợi vs thực tế (200.000đ tạm thời), tần suất tái hiện, network log/HAR, video, log backend, order total cuối.
 
## 3.3. Voucher đã dùng nhưng cache chưa cập nhật
 
**Expected behavior cần xác nhận ở bước nào?**

Cần BA/PO xác nhận kiểm tra tính hợp lệ (voucher đã dùng) diễn ra tại **bước Apply Voucher** (và vẫn kiểm tra lại ở Place Order). Backend là nguồn sự thật duy nhất; Apply phải kiểm tra theo dữ liệu mới nhất, không dựa vào cache cũ.
 
**Rủi ro:**
- UX: người dùng nhìn thấy "áp dụng thành công" và giá đã giảm, đến lúc đặt hàng mới bị từ chối → mất niềm tin, bỏ giỏ.
- Data consistency: dữ liệu giữa thiết bị/service không đồng bộ; nguy cơ voucher bị dùng hai lần nếu Place Order không kiểm tra lại (conflict giữa hai thiết bị).
- Giá hiển thị trước và sau Place Order khác nhau, có thể gây khiếu nại.

**Scenario cần thêm:**
- **BF-04 (áp voucher):** Apply NEW100 trên thiết bị B sau khi đã dùng trên thiết bị A (cache cũ) → kỳ vọng bị từ chối ngay khi Apply (đã xác nhận).
- **BF-04:** Apply trên hai thiết bị gần như đồng thời khi voucher còn một lượt dùng.
- **BF-06 (đặt hàng):** Place Order với voucher đã bị dùng nơi khác → order không được tạo (hoặc tạo không voucher, theo rule), thông báo rõ ràng, tổng tiền cập nhật.
- **BF-06:** Đặt hai order song song dùng cùng voucher 1 lượt → chỉ một thành công.

**Có nên biến thành regression test không?** 
Có. Lý do: (1) liên quan tiền/giá nên impact cao; (2) lỗi thuộc loại dễ tái phát khi thay đổi caching hoặc logic voucher; (3) đã từng thực sự xảy ra (defect có thật); (4) có thể tự động hóa. Đưa vào regression suite với test data rõ ràng (user đã dùng NEW100, hai session).
 
## 3.4. Cart thay đổi sau khi voucher đã apply
 
**Tình huống:** Cart 650.000đ, NEW100 đã apply; xóa item còn 480.000đ (< 500.000đ).
 
**Requirement hiện tại đủ rõ chưa?** **Chưa.** Requirement chỉ nêu điều kiện khi áp dụng, không nêu điều gì xảy ra khi cart thay đổi sau đó. Có ba hướng: (a) tự gỡ voucher và thông báo; (b) giữ voucher nhưng không áp dụng (hiển thị lý do) và tự áp lại nếu cart đủ điều kiện; (c) chặn thao tác xóa. Tester **không tự chọn** expected result.
 
**Question/Assumption:**
- **Question:** Khi subtotal giảm xuống dưới 500.000đ sau khi đã áp NEW100, hệ thống nên: tự gỡ voucher / giữ nhưng không áp dụng / chặn xóa? Có hiển thị thông báo không? Voucher có bị tính là "đã dùng" không? Nếu cart tăng lại ≥ 500.000đ có tự áp dụng lại?
- **Assumption:** Hệ thống tự gỡ voucher và hiển thị thông báo; total tính lại theo subtotal mới (480.000đ + ship). Test case gắn nhãn "Pending BA confirmation" và không dùng làm tiêu chí pass/fail chính thức.

**Artifact (Template 1–7) bị ảnh hưởng khi BA xác nhận rule mới:** Requirement Analysis / Question log, Business Flow (BF-04 thêm nhánh thay đổi cart), Scenario list, Test Case chi tiết (biên 500.000đ sau khi sửa cart), Traceability Matrix, Test Data, Risk map, và Test Execution Summary (nếu đã chạy trước đó). Nếu mapping của team khác, cập nhật mọi template có chứa BF-04 hoặc rule voucher.
 
## 3.5. Hotfix voucher trước giờ release (30 phút)
 
**Chọn gì trước?** **Sanity trước, sau đó Smoke ngắn trên build release candidate, rồi Regression theo risk với thời gian còn lại.**
- Sanity vì thay đổi hẹp (lỗi voucher hết hạn): xác nhận nhanh bản sửa đúng và không gây lỗi rõ ràng quanh khu vực đó.
- Smoke để đảm bảo build ổn định (login, cart, checkout chạy được) trước khi đầu tư test sâu.
- Không đủ thời gian chạy full regression nên chọn theo risk.

**Flow khác ngoài BF-04 có thể bị ảnh hưởng:** Tính giá/total ở cart và checkout, BF-06 (Place Order — kiểm tra lại voucher), tính phí ship (free shipping dựa trên total sau voucher), Order History/hiển thị đơn dùng voucher, và (nếu có) hoàn voucher khi hủy đơn.
 
**Test set theo risk (≤ 6 scenario):**
1. NEW100 còn hạn + cart đủ điều kiện → áp dụng đúng.
2. NEW100 hết hạn → bị từ chối với thông báo đúng.
3. NEW100 vào đúng ngày/giờ biên hết hạn → hành vi đúng theo quy ước.
4. Voucher hợp lệ khác (hoặc không voucher) → không bị ảnh hưởng.
5. Apply voucher → Place Order → order total và order record đúng.
6. Voucher + free shipping (biên total sau voucher) → phí ship đúng.

**Remaining risk trong Test Execution Summary:** Chưa chạy full regression; các flow chỉ test mức smoke/sanity; chưa test hiệu năng và đồng thời (double-click, race); chưa test đa thiết bị/cache; chưa test với dữ liệu thật/múi giờ; UAT chưa có; nêu rõ phần nào chưa kiểm và đề xuất người quyết định release chấp nhận hay không.
 
## 3.6. Notification fail sau khi order thành công
 
**Order có fail không?** Chưa thể tự kết luận. Việc gửi email là side effect; nó có coi là một phần của giao dịch hay không là **business decision**. Tester chưa thể tự quyết vì requirement chưa nói rõ: (a) email là critical hay non-critical, (b) có phải rollback order, (c) có retry hay thông báo cho user.
 
**Câu hỏi cho BA/PO:**
1. Nếu email gửi lỗi, order vẫn giữ trạng thái thành công, vậy có đúng không?
2. Có retry tự động không? Bao nhiêu lần, cách nhau bao lâu?
3. UI có cần thông báo cho user rằng email chưa gửi được không? Cách khác để user xác nhận đơn (Order History, SMS)?
4. Lỗi email có cần ghi log/alert cho vận hành không?
5. Email xác nhận có nội dung/pháp lý quan trọng (hóa đơn) hay chỉ mang tính thông tin?

**Nếu email là side effect không critical:**
- *Expected:* Order vẫn thành công; UI hiển thị confirmation; notification được ghi nhận trạng thái Failed và đưa vào retry; có log lỗi; không tạo duplicate order khi retry.
- *Defect severity:* Nếu hệ thống hoạt động đúng như trên → không phải defect. Nếu **order bị rollback hoặc user thấy lỗi dù order đã tạo** → Severity cao (sai dữ liệu/giao dịch). Nếu **không retry/không log** → Severity low-medium (Priority theo ý kiến BA). Nếu email chứa thông tin bắt buộc (hóa đơn) thì Severity tăng.
---
 
# PHẦN 4 — INTERVIEW QUESTIONS
 
**1. Testing Level, Type, Approach khác nhau thế nào?**
- *Level*: kiểm thử ở **phạm vi/giai đoạn nào** (Unit, Integration, System, Acceptance).
- *Type*: kiểm thử **cái gì** (Functional, Non-functional như performance/security/usability, Regression…).
- *Approach*: kiểm thử **bằng cách nào/kiến thức gì** (Black-box, White-box, Gray-box; scripted hay exploratory).
- *Ví dụ:* nhập NEW100 cho cart 520.000đ rồi kiểm tra total hiển thị → Level: System; Type: Functional; Approach: Black-box (scripted).

**2. Unit và Integration Testing khác nhau về phạm vi và loại defect dễ phát hiện như thế nào?**
- Unit test kiểm từng hàm/class độc lập (có thể mock), dev thường thực hiện; bắt lỗi logic cục bộ, sai biên, sai tính toán (ví dụ hàm tính discount sai).
- Integration test kiểm việc các module/service phối hợp với nhau(API ↔ DB, cart ↔ voucher service); bắt lỗi giao tiếp: sai format dữ liệu, sai contract, sai thứ tự gọi, lỗi transaction.

**3. System Testing khác Acceptance/UAT ở mục tiêu nào?**
- System: kiểm hệ thống hoàn chỉnh so với **requirement/specification**, do QA thực hiện, tìm lỗi trước khi giao.
- UAT: người dùng/khách hàng/business kiểm xem hệ thống có **đáp ứng nhu cầu thực tế** và sẵn sàng đưa vào sử dụng không; mục tiêu là quyết định chấp nhận, không chỉ tìm lỗi.

**4. Alpha vs Beta**
- Alpha: người dùng nội bộ/đại diện, môi trường kiểm soát (gần dev/staging), feedback để sửa lỗi lớn trước khi ra ngoài.
- Beta: người dùng thật, môi trường thực tế (production-like), feedback về trải nghiệm, tương thích và lỗi chỉ lộ ra ngoài thực tế.

**5. Functional vs Non-functional**
- Functional: "Hệ thống **làm đúng việc** theo requirement không?" (voucher giảm đúng tiền, đặt hàng tạo order).
- Non-functional trả lời: "Hệ thống **làm tốt đến mức nào**?" (nhanh, an toàn, dễ dùng, ổn định, tương thích — ví dụ Apply Voucher phản hồi dưới 2 giây khi 500 người dùng).

**6. Smoke vs Sanity**
- Smoke: trigger khi có **build mới**; mục tiêu: build có đủ ổn định để test tiếp không (login, mở giỏ, checkout).
- Sanity: trigger sau **thay đổi/bug fix cụ thể**; mục tiêu: bản sửa có đúng và không làm tổn hại đến khu vực liên quan không.

**7. Regression — chọn scope & vì sao không phải lúc nào cũng chạy toàn bộ suite**
- Dựa trên: vùng code/flow bị thay đổi, các phụ thuộc, mức độ rủi ro/độ quan trọng business, lịch sử lỗi, thời gian có sẵn.
- Không chạy toàn bộ vì tốn thời gian/chi phí, phản hồi chậm; chạy theo risk để đạt giá trị cao nhất; full regression thường chạy ở mốc quan trọng (trước release).

**8. Exploratory có phải tùy hứng không?**
- Không. Đó là kiểm thử có cấu trúc: vừa học vừa thiết kế vừa thực thi, có mục tiêu và ghi chép. Charter tốt gồm: **Mục tiêu** (khám phá gì), **Phạm vi/khu vực**, **Tài nguyên/dữ liệu/công cụ**, **Time-box**, **Cách ghi nhận** (notes, bug, câu hỏi). Ví dụ: "Khám phá luồng Apply Voucher dưới mạng chậm để tìm lỗi trạng thái; 45 phút; dùng throttle 3G; ghi lại network log".

**9. Black-box, White-box, Gray-box**
- Black-box: không biết nội bộ, test theo input/output và requirement.
- White-box: biết code/cấu trúc, test theo path/branch/condition.
- Gray-box: biết một phần (DB schema, API, log) để thiết kế test tốt hơn và kiểm tra trạng thái nội bộ.

**10. Static vs Dynamic**
- Static: **không chạy** phần mềm; review requirement/code/design, walkthrough, static analysis (ví dụ review requirement phát hiện thiếu rule hủy lịch).
- Dynamic: **chạy** phần mềm với dữ liệu để quan sát hành vi (ví dụ chạy test voucher hết hạn).

**11. Vì sao ghi Question/Assumption thay vì tự đặt expected result?**
- Expected result tự đoán có thể sai so với ý đồ business → test pass/fail giả, defect bị tranh cãi, rework. Ghi Question/Assumption làm lộ khoảng trống, người chịu trách nhiệm trả lời, giữ minh bạch và truy vết. Ví dụ: không tự quyết giới hạn thời gian hủy lịch; hỏi BA rồi mới viết expected result.

**12. Traceability Requirement → Flow → Scenario → Test Case**
- Khi requirement thay đổi, biết ngay flow/scenario/test case nào cần cập nhật (impact analysis), tránh sót test hoặc test thừa; chứng minh coverage; hỗ trợ chọn regression theo thay đổi; giúp báo cáo.

**13. Priority vs Severity**
- Severity: mức độ **ảnh hưởng kỹ thuật/hệ thống** của lỗi. 
- Priority: **thứ tự cần sửa** theo business.
- Ví dụ: sai chính tả logo trang chủ → Severity thấp, Priority cao . Crash ở tính năng ít dùng → Severity cao, Priority thấp hơn.

**14. Evidence tốt gồm gì? Vì sao chỉ chụp một ảnh đôi khi chưa đủ?**
- Gồm: bước tái hiện, dữ liệu test, môi trường/build, ảnh/video, log (client/server), network request/response, bản ghi DB, thời điểm, tần suất. Một ảnh chỉ thấy kết quả tại một thời điểm, không thấy nguyên nhân, trạng thái backend hay chuỗi sự kiện (ví dụ discount 200.000đ chỉ hiện vài giây).

**15. Build mới — test gì trước?**
- Smoke trước (build chạy được, luồng chính hoạt động). Sau đó ưu tiên theo risk (khả năng xảy ra + tác động): khu vực vừa thay đổi, flow liên quan tiền/dữ liệu (checkout, voucher, thanh toán), vùng nhiều lỗi, flow người dùng dùng nhiều nhất. Phần còn lại test sau theo thời gian.
---
 
## 4.1. Mini Interview Cases
 
### Case A — Classification
**Scenario:** nhập NEW100 cho cart 500.000đ, hệ thống tính lại total trên UI.
- **Level:** System (UI + logic giá + backend); có thể coi là Integration nếu tách riêng kiểm tra tương tác UI và API giá.
- **Type:** Functional (kiểm tra đúng discount/total), thêm tính **boundary** vì 500.000đ là biên của điều kiện.
- **Approach:** Black-box (thiết kế từ requirement, phân tích giá trị ngưỡng), dạng scripted.
- **Giải thích:** đây là kiểm tra hệ thống đã tích hợp theo đầu vào/đầu ra; mục tiêu là xác nhận rule tại biên (≥ hay >) — cần BA xác nhận expected nếu requirement chưa rõ.
### Case B — Ambiguous Requirement
**Requirement:** "Voucher chỉ áp dụng cho khách hàng mới."
 
Câu hỏi cần làm rõ:
1. **Khách hàng mới** định nghĩa thế nào: chưa từng có order, chưa từng đặt thành công, hay mới đăng ký trong N ngày?
2. Order bị hủy/thất bại/hoàn tiền thì voucher có hoàn lại cho khách hàng đó không?
3. Xác định theo tài khoản, email, số điện thoại, thiết bị hay địa chỉ giao hàng cho khách hàng để chống việc tạo nhiều tài khoản để nhận voucher?
4. Khách vãng lai (guest) có được dùng không?
5. Kiểm tra vào bước "Apply voucher" hay "Place Order"?
6. Thông báo lỗi hiển thị thế nào khi không đủ điều kiện?
### Case C — Release Decision
Smoke pass, 20 regression test có 1 fail liên quan Order History, UAT chưa chạy. 

**Test Summary cần báo cáo:**
- Phạm vi test: build, môi trường, số lượng test đã chạy (Smoke pass; Regression 20 test: 19 pass, 1 fail).
- Chi tiết defect: ID, mô tả, mức Severity/Priority đề xuất, flow ảnh hưởng (Order History), tái hiện được không, evidence.
- Đánh giá tác động: người dùng/nghiệp vụ nào bị ảnh hưởng (xem lịch sử đơn), có ảnh hưởng dữ liệu hay chỉ hiển thị.
- Remaining risk: UAT chưa chạy; phần chưa test; defect chưa sửa/chưa retest.
- Khuyến nghị: không tự quyết thay stakeholder — nêu rõ điều kiện (ví dụ: fix và retest lỗi Order History, chạy UAT hoặc chấp nhận rủi ro) để PO/quản lý quyết định.
### Case D — Defect Writing
 
- **Title:** Bấm "Place Order" hai lần liên tiếp tạo hai Order ID khác nhau cho cùng một giỏ hàng
- **Severity đề xuất:** **High/Critical** (tạo đơn trùng, ảnh hưởng tiền và tồn kho; Priority: High).
- **Preconditions:**
  - Đã đăng nhập tài khoản test (ghi rõ), cart có ≥ 1 sản phẩm, địa chỉ và phương thức thanh toán hợp lệ.
  - Build/môi trường: version 1.1, trình duyệt/thiết bị: Laptop Asus Vivobook.
- **Steps:**
  1. Mở trang Checkout với cart đã chuẩn bị.
  2. Bấm nút "Place Order".
  3. Ngay lập tức bấm "Place Order" lần thứ hai (trước khi trang chuyển).
  4. Mở Order History và xem các order vừa tạo.
- **Expected:** Chỉ một order được tạo (một Order ID); nút bị vô hiệu hóa sau lần bấm đầu; request thứ hai không tạo order mới (idempotent); tồn kho/voucher/thanh toán chỉ bị tác động một lần.
- **Actual:** Hệ thống tạo **hai** Order ID khác nhau cho cùng giỏ hàng; (nếu có) tồn kho/voucher/thanh toán bị trừ hai lần.
- **Evidence:** Ảnh/video hiển thị hai Order ID; network log/HAR có hai request POST /orders kèm thời điểm và response; log backend; hai bản ghi order trong DB (ID, thời điểm tạo, tổng tiền); số lần tái hiện (3/3).



