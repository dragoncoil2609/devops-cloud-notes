# 📘 HƯỚNG DẪN TRIỂN KHAI FRONTEND LÊN S3 + CLOUDFRONT (GIẢI PHÁP SINGLE DOMAIN CHUYÊN NGHIỆP)

Hạ tầng Backend và Database đã hoạt động cực kỳ mượt mà. Bước tiếp theo của chúng ta là đưa Frontend lên môi trường Production và kết nối hoàn chỉnh với Backend thông qua **Giải pháp Single Domain (Tên miền duy nhất)**.

---

## 🌐 TẠI SAO PHẢI DÙNG GIẢI PHÁP SINGLE DOMAIN?

Thay vì chạy Frontend ở một đường dẫn S3 công cộng và Backend ở đường dẫn Load Balancer (ALB), việc này sẽ gây ra 2 lỗi cực kỳ nghiêm trọng khiến trình duyệt chặn ứng dụng:
1.  **CORS (Cross-Origin Resource Sharing):** Sự khác biệt giữa tên miền Frontend và Backend khiến trình duyệt chặn các request gọi API.
2.  **Mixed Content:** Trình duyệt sẽ chặn toàn bộ API HTTP thường khi ứng dụng Frontend chạy trên HTTPS.

**Giải pháp:** Chúng ta sử dụng **AWS CloudFront (CDN)** làm "người gác cổng" duy nhất đại diện cho tên miền `minie-ecommercehoangdeptraisieucaovutru.software`:
*   Mọi truy cập thông thường (như trang chủ `/`, `/products`, `/cart`,...) ──> Chuyển tới **S3 Bucket** chứa Frontend.
*   Mọi request gọi API (đường dẫn `/api/*`) ──> Định tuyến trực tiếp tới **Application Load Balancer (ALB)** chứa Backend.
*   **Kết quả:** Hệ thống đạt hiệu năng tải tĩnh cực cao, bảo mật SSL (HTTPS) đồng bộ trên 1 tên miền duy nhất, loại bỏ hoàn toàn cấu hình CORS rườm rà!

---

## 📦 BƯỚC 1: TẠO S3 BUCKET CHỨA CODE FRONTEND (BẢO MẬT TUYỆT ĐỐI)

1.  Vào AWS Console ──> Tìm dịch vụ **S3** ──> Click **Create bucket**.
2.  Thiết lập các thông số:
    *   **Bucket name:** `minie-frontend-production` (hoặc tên tùy chọn duy nhất của bạn).
    *   **Region:** Chọn cùng khu vực với hệ thống của bạn (ví dụ: `us-east-1`).
    *   **Object Ownership:** Chọn **ACLs disabled (recommended)**.
    *   **Block Public Access settings for this bucket:** **TÍCH CHỌN** **Block all public access** (Bật tính năng chặn hoàn toàn truy cập công cộng. Chúng ta sẽ dùng CloudFront OAC để truy cập bảo mật, không cần mở public công khai).
3.  Click **Create bucket**.

---

## 🛡️ BƯỚC 2: TẠO CLOUDFRONT DISTRIBUTION (BỘ PHÂN PHỐI CDN GÁC CỔNG)

1.  Vào AWS Console ──> Tìm dịch vụ **CloudFront** ──> Click **Create distribution**.
2.  **Cấu hình Origin 1 (Frontend S3):**
    *   **Origin domain:** Chọn đúng S3 bucket `minie-frontend-production` bạn vừa tạo ở Bước 1.
    *   **Origin access:** Tích chọn **Origin access control settings (recommended)**.
    *   **Origin access control:** Click nút **Create control setting** ──> Giữ nguyên các giá trị mặc định ──> Click **Create**.
3.  **Cấu hình Default Cache Behavior (Mặc định gửi vào S3):**
    *   **Viewer protocol policy:** Chọn **Redirect HTTP to HTTPS** (Tự động chuyển từ HTTP thường sang HTTPS bảo mật).
    *   **Allowed HTTP methods:** Chọn **GET, HEAD, OPTIONS**.
4.  **Cấu hình Settings (Thông tin Tên miền & Chứng chỉ SSL):**
    *   **Price class:** Chọn *Use all edge locations* (hoặc *Use only North America and Europe* để tiết kiệm).
    *   **Alternate domain name (CNAME):** Click **Add item** và điền 2 dòng tên miền của bạn:
        *   Dòng 1: `minie-ecommercehoangdeptraisieucaovutru.software`
        *   Dòng 2: `www.minie-ecommercehoangdeptraisieucaovutru.software`
    *   **Custom SSL certificate:** Tích chọn chứng chỉ SSL Wildcard mà chúng ta đã đăng ký thành công trên ACM ở bước trước (dạng `*.minie-ecommercehoangdeptraisieucaovutru.software`).
    *   **Default root object:** Điền `index.html`.
5.  Click **Create distribution** ở dưới cùng.

---

## 🧬 BƯỚC 3: CẬP NHẬT BUCKET POLICY CHO S3 (CHO PHÉP CLOUDFRONT ĐỌC CODE)

1.  Sau khi tạo CloudFront thành công, ở trên cùng màn hình sẽ hiển thị một banner màu xanh lá thông báo: *"The S3 bucket policy needs to be updated..."*.
2.  Bạn click vào nút **Copy policy** ở banner đó.
3.  Quay lại trang dịch vụ **S3** ──> Click vào bucket `minie-frontend-production` ──> Chọn tab **Permissions**.
4.  Cuộn xuống phần **Bucket policy** ──> Click **Edit** ──> Dán (Paste) đoạn policy bạn vừa copy vào ô soạn thảo.
5.  Click **Save changes** để lưu lại. *(Bây giờ CloudFront đã có quyền đọc code tĩnh từ S3 một cách bảo mật)*.

---

## 🗺️ BƯỚC 4: THÊM ORIGIN BACKEND (ALB) VÀ ĐỊNH TUYẾN `/api/*`

*Mục đích: Dẫn các request API từ tên miền chính vào thẳng Load Balancer của Backend.*

1.  Quay lại trang chi tiết **CloudFront Distribution** vừa tạo ──> Chọn tab **Origins**.
2.  Click **Create origin** để thêm Backend:
    *   **Origin domain:** Tìm và chọn đường dẫn DNS của Load Balancer của bạn (dạng: `alb-production-xxxxxx.us-east-1.elb.amazonaws.com`).
    *   **Protocol:** Chọn **HTTP only** (Do ALB của chúng ta chạy cổng 80/3000 bình thường, CloudFront sẽ chịu trách nhiệm bọc HTTPS bảo mật bên ngoài).
    *   Giữ nguyên các thông số khác và click **Create origin**.
3.  Chọn tab **Behaviors** ──> Click **Create behavior** để tạo luật định tuyến cho API:
    *   **Path pattern:** Điền chính xác **`/api/*`**
    *   **Origin:** Chọn đúng Origin Load Balancer (ALB) bạn vừa tạo ở trên.
    *   **Viewer protocol policy:** Chọn **Redirect HTTP to HTTPS** (Đảm bảo mọi kết nối API đều đi qua cổng bảo mật HTTPS).
    *   **Allowed HTTP methods:** Chọn **GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE** *(Rất quan trọng! Backend bắt buộc phải có đầy đủ các phương thức này để xử lý việc đăng nhập, đặt hàng, thêm sửa xóa).*
    *   **Cache key and origin requests:**
        *   Tích chọn **Cache policy and origin request policy (recommended)**.
        *   **Cache policy:** Chọn **`CachingDisabled`** (Để CloudFront không lưu bộ nhớ đệm cho API Backend, tránh việc bị cache dữ liệu nhầm lẫn).
        *   **Origin request policy:** Chọn **`AllViewerExceptHostHeader`** *(Cực kỳ quan trọng! Giúp chuyển tiếp đầy đủ Cookie đăng nhập, Authorization Header và các tham số tìm kiếm từ người dùng tới thẳng Backend).*
4.  Click **Create behavior** ở cuối trang.

---

## 🔄 BƯỚC 5: CẤU HÌNH TRÁNH LỖI REFRESH TRANG (DÀNH CHO SPA REACT/VITE)

Khi người dùng nhấn F5 làm mới trang ở một đường dẫn con (ví dụ: `/cart`, `/profile`), S3 sẽ báo lỗi `403 Forbidden` vì không tìm thấy file vật lý tên là `/cart` hay `/profile` trên S3. Chúng ta cần cấu hình CloudFront tự động chuyển các lỗi này về file gốc `/index.html`:

1.  Tại trang chi tiết CloudFront Distribution, chọn tab **Error pages**.
2.  Click **Create custom error response**.
3.  Cấu hình lỗi **403: Forbidden**:
    *   **HTTP error code:** Chọn **403: Forbidden**.
    *   **Customize error response:** Chọn **Yes**.
    *   **Response page path:** Điền **`/index.html`**
    *   **HTTP Response code:** Chọn **200: OK**.
    *   Click **Create**.
4.  Click tiếp **Create custom error response** lần nữa để cấu hình lỗi **404: Not Found**:
    *   **HTTP error code:** Chọn **404: Not Found**.
    *   **Customize error response:** Chọn **Yes**.
    *   **Response page path:** Điền **`/index.html`**
    *   **HTTP Response code:** Chọn **200: OK**.
    *   Click **Create**.

---

## 🛜 BƯỚC 6: TRỎ TÊN MIỀN TỪ NAME.COM VỀ CLOUDFRONT

1.  Tại trang chi tiết CloudFront Distribution, chọn tab **Details**.
2.  Tìm và copy đường dẫn **Distribution domain name** (dạng: `xxxxxxxxxxxxxx.cloudfront.net`).
3.  Đăng nhập vào trang quản lý tên miền **Name.com** (nơi bạn mua tên miền).
4.  Vào phần cấu hình DNS (DNS Zone Editor) và thêm/chỉnh sửa 2 bản ghi:
    *   **Bản ghi 1 (Cho tên miền chính `@`):**
        *   **Host:** `@` (hoặc để trống tên miền gốc).
        *   **Type:** Chọn **ANAME** hoặc **ALIAS**.
        *   **Answer / Value:** Dán đường dẫn CloudFront đã copy ở trên (ví dụ: `xxxxxxxxxxxxxx.cloudfront.net`).
    *   **Bản ghi 2 (Cho tên miền phụ `www`):**
        *   **Host:** `www`
        *   **Type:** Chọn **CNAME**.
        *   **Answer / Value:** Dán đường dẫn CloudFront `xxxxxxxxxxxxxx.cloudfront.net`.
5.  Click **Save changes** để lưu lại.

---

## 📦 BƯỚC 7: BUILD CODE FRONTEND VÀ UPLOAD LÊN S3 BUCKET

1.  Trên máy tính của bạn, mở thư mục dự án Frontend `d:\DOANTOTNGHIEP\FE\mini-e_fe_web`.
2.  Mở file `.env` và đảm bảo đường dẫn API đã chính xác tuyệt đối như sau:
    ```env
    VITE_API_BASE_URL=https://minie-ecommercehoangdeptraisieucaovutru.software/api
    ```
3.  Mở Terminal tại thư mục Frontend này và chạy lệnh biên dịch dự án:
    ```bash
    npm run build
    ```
    *Lệnh này sẽ tạo ra một thư mục mới tên là **`dist`** chứa file tĩnh và toàn bộ mã nguồn Frontend.*
4.  Truy cập **S3 Console** ──> Click chọn bucket **`minie-frontend-production`** đã tạo ở Bước 1.
5.  Click nút **Upload**.
6.  **Lưu ý:** Bạn mở thư mục `dist` trên máy tính ra, hãy bôi đen **toàn bộ các file và thư mục con NẰM TRONG thư mục `dist`** (bao gồm file `index.html`, thư mục `assets`...) rồi kéo thả tất cả vào trang Upload của S3. 
    *(Không kéo nguyên cả thư mục cha `dist` vào, chỉ kéo các tài nguyên bên trong nó).*
7.  Click **Upload** và chờ quá trình tải lên hoàn tất!

---

### 🎉 KẾT QUẢ ĐẠT ĐƯỢC:
Sau khi hoàn thành đầy đủ các bước, bạn truy cập trực tiếp tên miền chính thức của mình:
👉 **`https://minie-ecommercehoangdeptraisieucaovutru.software`**

Hệ thống sẽ hoạt động thông suốt từ giao diện Frontend tĩnh cho đến các tác vụ API động kết nối trực tiếp vào ECS/EC2 Backend và lưu trữ vào RDS Database vô cùng an toàn và chuyên nghiệp!
