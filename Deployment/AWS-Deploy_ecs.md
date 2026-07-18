# 📘 HƯỚNG DẪN CHI TIẾT TRIỂN KHAI TOÀN DIỆN TRÊN AWS ECS FARGATE (STEP-BY-STEP TỪ A - Z)

Tài liệu hướng dẫn từng bước cấu hình trực quan và dòng lệnh để triển khai hoàn chỉnh ứng dụng E-commerce đa lớp trên **AWS ECS Fargate**, giao tiếp với **RDS MySQL** và **Application Load Balancer (ALB)**, kết hợp **S3** & **CloudFront** cho Frontend.

---

## 🔒 BƯỚC 0: PHÂN QUYỀN HỆ THỐNG (IAM)

### 1. Tạo IAM User Quản trị viên
1. Vào AWS Console ──> Tìm dịch vụ **IAM** ──> Chọn **Users** ──> Click **Create user**.
2. **User name:** Nhập tên user quản trị (Ví dụ: `<IAM_ADMIN_USER>`).
3. Tích chọn **Provide user access to the AWS Management Console**.
4. Chọn **I want to create an IAM user**.
5. **Console password:** Chọn **Custom password** và điền mật khẩu của bạn.
6. Tích chọn **Users must create a new password at next sign-in (Recommended)**.
7. Click **Next**.
8. **Set quyền:** Chọn ô **Attach policies directly** ──> Tìm quyền `AdministratorAccess` và tích chọn.
9. Click **Next** ──> Click **Create user**.

### 2. Thiết lập MFA (Multi-Factor Authentication)
1. Chọn IAM User vừa tạo ──> Chuyển sang tab **Security credentials**.
2. Tìm mục **Multi-factor authentication (MFA)** ──> Click **Assign MFA device**.
3. **Device name:** Gõ tên gợi nhớ (Ví dụ: `mfa-<IAM_ADMIN_USER>`).
4. Chọn **Authenticator app** ──> Click **Next**.
5. Dùng ứng dụng Authenticator (Google Authenticator, Authy...) trên điện thoại quét mã QR và nhập 2 mã xác thực liên tiếp để xác nhận.

### 3. Tạo Role `ecsTaskExecutionRole` (Quyền chạy Task)
Role này cấp quyền cho máy chủ ảo của AWS (ECS Agent) kéo Docker Image từ ECR/Docker Hub và truy cập Secrets Manager lấy thông tin cấu hình nhét vào container.
1. Ở menu bên trái IAM, chọn **Roles** ──> Click **Create role**.
2. **Trusted entity type:** Chọn **AWS service**.
3. **Use case:** Tìm và chọn **Elastic Container Service** ──> Chọn **Elastic Container Service Task** (Lưu ý: Chọn đúng use case có chữ **Task**).
4. Click **Next**.
5. **Add quyền:**
   * Tìm và tích chọn: `AmazonECSTaskExecutionRolePolicy`
   * Tìm và tích chọn: `SecretsManagerReadWrite`
6. **Role name:** Đặt tên là `<ECS_TASK_EXECUTION_ROLE_NAME>` (Ví dụ: `ecsTaskExecutionRole`).
7. Click **Create role**.

### 4. Tạo Role `ecsTaskRole` (Quyền của Ứng dụng)
Role này cấp quyền trực tiếp cho mã nguồn ứng dụng (code chạy bên trong container) để tương tác với các dịch vụ AWS khác (như upload file lên S3).
1. Trước hết, tạo Policy truy cập S3: Vào IAM ──> Chọn **Policies** ──> Click **Create policy**.
2. Chọn tab **JSON** trong Policy editor, xóa nội dung cũ và dán đoạn mã sau:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "*"
        }
    ]
}
```
*(Lưu ý bảo mật: Trong môi trường thực tế, nên thay `"Resource": "*"` bằng ARN cụ thể của S3 Buckets).*
3. Click **Next** ──> **Policy name:** Đặt tên là `<S3_ACCESS_POLICY_NAME>` (Ví dụ: `EcomS3AccessPolicy`).
4. Click **Create policy**.
5. Tiến hành tạo Role tương tự bước 3:
   * **Use case:** Chọn **Elastic Container Service Task**.
   * **Add quyền:** Tìm và gắn policy `<S3_ACCESS_POLICY_NAME>` vừa tạo ở trên.
   * **Role name:** Đặt tên là `<ECS_TASK_ROLE_NAME>` (Ví dụ: `ecsTaskRole`).
   * Click **Create role**.

---

## 🛜 BƯỚC 1: XÂY DỰNG MẠNG LƯỚI BẢO MẬT (VPC)

### 1. Khởi tạo VPC
1. Vào dịch vụ VPC ──> Chọn **VPCs** ──> Click **Create VPC**.
2. Chọn **VPC only** để thiết lập thủ công từng bước.
3. **Name tag:** Gõ `<VPC_NAME>` (Ví dụ: `VPC-Lab`).
4. **IPv4 CIDR block:** Gõ `<VPC_CIDR>` (Ví dụ: `10.0.0.0/16`).
5. Click **Create VPC**.

### 2. Thiết lập Internet Gateway (IGW)
IGW là cánh cổng chính để kết nối VPC nội bộ ra thế giới Internet bên ngoài.
1. Chọn **Internet gateways** ──> Click **Create internet gateway**.
2. **Name tag:** Gõ `<IGW_NAME>` (Ví dụ: `IGW-Lab`) ──> Click **Create**.
3. Sau khi tạo xong, click **Actions** ──> Chọn **Attach to VPC**.
4. Chọn `<VPC_NAME>` từ danh sách ──> Click **Attach internet gateway**.

### 3. Khởi tạo Phân khu Public (Public Subnets)
Phân khu này sẽ chứa NAT Gateways và Application Load Balancer (ALB).
1. Chọn **Subnets** ──> Click **Create subnet**.
2. **VPC ID:** Chọn `<VPC_NAME>`.
3. **Subnet 1:**
   * **Subnet name:** `<PUBLIC_SUBNET_1_NAME>` (Ví dụ: `Nat-Subnet-1`).
   * **Availability Zone:** Chọn `<AZ_1>` (Ví dụ: `us-east-1a`).
   * **IPv4 CIDR block:** `<PUBLIC_SUBNET_1_CIDR>` (Ví dụ: `10.0.0.0/24`).
4. Click **Add new subnet** để tạo Subnet Public thứ 2 (dự phòng):
   * **Subnet name:** `<PUBLIC_SUBNET_2_NAME>` (Ví dụ: `Nat-Subnet-2`).
   * **Availability Zone:** Chọn `<AZ_2>` (Ví dụ: `us-east-1b`).
   * **IPv4 CIDR block:** `<PUBLIC_SUBNET_2_CIDR>` (Ví dụ: `10.0.1.0/24`).
5. Click **Create subnet**.

### 4. Định tuyến cho Public Subnets
1. Chọn **Route tables** ──> Click **Create route table**.
2. **Name:** Gõ `<PUBLIC_ROUTE_TABLE_NAME>` (Ví dụ: `IGW-Route-Table`).
3. **VPC:** Chọn `<VPC_NAME>` ──> Click **Create**.
4. **Trỏ đường ra Internet:** Chọn tab **Routes** ──> Click **Edit routes** ──> Click **Add route**.
   * **Destination:** `0.0.0.0/0`
   * **Target:** Chọn **Internet Gateway** ──> Chọn `<IGW_NAME>`.
5. Click **Save changes**.
6. Chọn tab **Subnet associations** ──> Click **Edit subnet associations** ──> Tích chọn `<PUBLIC_SUBNET_1_NAME>` và `<PUBLIC_SUBNET_2_NAME>` ──> Click **Save**.

### 5. Khởi tạo NAT Gateways (Cấp Internet một chiều cho Private Subnets)
NAT Gateway giúp ứng dụng ở Private Subnets gọi ra ngoài Internet (để kéo Docker Image, gọi API thanh toán...) nhưng chặn chiều ngược lại từ ngoài Internet xâm nhập trực tiếp.
1. Chọn **NAT gateways** ──> Click **Create NAT gateway**.
2. **Name:** Gõ `<NAT_GATEWAY_1_NAME>` (Ví dụ: `Nat-Lab-1`).
3. **Subnet:** Chọn `<PUBLIC_SUBNET_1_NAME>`.
4. **Elastic IP allocation ID:** Click **Allocate Elastic IP** để AWS cấp IP tĩnh công cộng.
5. Click **Create NAT gateway**.
6. Lặp lại tương tự để tạo NAT Gateway thứ 2:
   * **Name:** `<NAT_GATEWAY_2_NAME>` (Ví dụ: `Nat-Lab-2`).
   * **Subnet:** Chọn `<PUBLIC_SUBNET_2_NAME>`.
   * **Elastic IP:** Click **Allocate Elastic IP** ──> Click **Create**.

### 6. Khởi tạo Phân khu Private cho Application (Private App Subnets)
1. Chọn **Subnets** ──> Click **Create subnet** ──> Chọn `<VPC_NAME>`.
2. **Subnet 1:**
   * **Subnet name:** `<PRIVATE_APP_SUBNET_1_NAME>` (Ví dụ: `App-Subnet-1`).
   * **Availability Zone:** Chọn `<AZ_1>`.
   * **IPv4 CIDR block:** `<PRIVATE_APP_SUBNET_1_CIDR>` (Ví dụ: `10.0.10.0/22`).
3. Click **Add new subnet** để tạo Subnet App thứ 2:
   * **Subnet name:** `<PRIVATE_APP_SUBNET_2_NAME>` (Ví dụ: `App-Subnet-2`).
   * **Availability Zone:** Chọn `<AZ_2>`.
   * **IPv4 CIDR block:** `<PRIVATE_APP_SUBNET_2_CIDR>` (Ví dụ: `10.0.14.0/22`).
4. Click **Create subnet**.

### 7. Định tuyến cho Private App Subnets
Chúng ta tạo 2 Route Table riêng biệt cho 2 Subnet App để điều phối traffic qua 2 NAT Gateway độc lập.
1. **Tạo Route Table cho App 1:**
   * Vào **Route tables** ──> Click **Create**.
   * **Name:** Gõ `<APP_ROUTE_TABLE_1_NAME>` (Ví dụ: `App-1-Route-Table`).
   * **VPC:** Chọn `<VPC_NAME>` ──> Click **Create**.
   * **Edit routes** ──> Add route: Destination `0.0.0.0/0` ──> Target chọn NAT Gateway `<NAT_GATEWAY_1_NAME>` ──> Save.
   * **Subnet associations** ──> Edit ──> Tích chọn `<PRIVATE_APP_SUBNET_1_NAME>` ──> Save.
2. **Tạo Route Table cho App 2:**
   * Lặp lại y hệt bước trên: Tạo `<APP_ROUTE_TABLE_2_NAME>` (Ví dụ: `App-2-Route-Table`), trỏ route `0.0.0.0/0` về `<NAT_GATEWAY_2_NAME>`, và gắn với `<PRIVATE_APP_SUBNET_2_NAME>`.

### 8. Khởi tạo Phân khu Private cho Database (Private DB Subnets)
Database cần cô lập hoàn toàn, không có route hướng ra NAT Gateway hay Internet Gateway.
1. Vào **Subnets** ──> Click **Create subnet** ──> Chọn `<VPC_NAME>`.
2. **DB Subnet 1:** Name `<PRIVATE_DB_SUBNET_1_NAME>` (Ví dụ: `DB-Subnet-1`), AZ `<AZ_1>`, CIDR `<PRIVATE_DB_SUBNET_1_CIDR>` (Ví dụ: `10.0.2.0/24`).
3. **DB Subnet 2:** Click Add new, Name `<PRIVATE_DB_SUBNET_2_NAME>` (Ví dụ: `DB-Subnet-2`), AZ `<AZ_2>`, CIDR `<PRIVATE_DB_SUBNET_2_CIDR>` (Ví dụ: `10.0.3.0/24`).
4. Click **Create subnet**.
5. **Cấu hình bảng định tuyến DB:**
   * Tạo Route Table mới tên `<DB_ROUTE_TABLE_NAME>` (Ví dụ: `DB-Route-Table`) cho `<VPC_NAME>`.
   * Gắn nó với cả 2 subnet `<PRIVATE_DB_SUBNET_1_NAME>` và `<PRIVATE_DB_SUBNET_2_NAME>`.
   * *Lưu ý:* Tuyệt đối không thêm bất kỳ route nào ra Internet hay NAT tại bảng này.

---

## 🔒 BƯỚC 2: THIẾT LẬP TƯỜNG LỬA (SECURITY GROUPS CHAINING)

Chúng ta cấu hình bảo mật liên kết nối tiếp nhau để đảm bảo an toàn tối đa cho hệ thống.

### 1. Nhóm 1: `<SG_ALB_NAME>` (Tường lửa của Load Balancer)
Nhiệm vụ: Nhận request công cộng cổng 80/443 từ bên ngoài Internet.
1. Vào dịch vụ VPC ──> Chọn **Security Groups** ──> Click **Create security group**.
2. **Security group name:** `<SG_ALB_NAME>` (Ví dụ: `ALB-SG`).
3. **Description:** Allow public HTTP/HTTPS traffic to ALB.
4. **VPC:** Chọn `<VPC_NAME>`.
5. **Inbound rules (Luật vào):** Add 2 rules:
   * Rule 1: Type `HTTP` | Source `Anywhere-IPv4 (0.0.0.0/0)`.
   * Rule 2: Type `HTTPS` | Source `Anywhere-IPv4 (0.0.0.0/0)`.
6. Click **Create**.

### 2. Nhóm 2: `<SG_ECS_APP_NAME>` (Tường lửa của Container Backend)
Nhiệm vụ: Chỉ nhận traffic đi từ Load Balancer (ALB) chuyển vào, chặn mọi truy cập trực tiếp khác.
1. Click **Create security group**.
2. **Security group name:** `<SG_ECS_APP_NAME>` (Ví dụ: `App-ECS-SG`).
3. **Description:** Allow traffic from ALB only.
4. **VPC:** Chọn `<VPC_NAME>`.
5. **Inbound rules (Luật vào):**
   * Type: Chọn `Custom TCP` | Port range: `<CONTAINER_PORT>` (Ví dụ: `3000`) | Source: Chọn **Custom**, click vào ô tìm kiếm bên cạnh và chọn nhóm bảo mật `<SG_ALB_NAME>`.
6. Click **Create**.

### 3. Nhóm 3: `<SG_RDS_DB_NAME>` (Tường lửa của Database)
Nhiệm vụ: Chỉ nhận truy cập từ các Container ứng dụng.
1. Click **Create security group**.
2. **Security group name:** `<SG_RDS_DB_NAME>` (Ví dụ: `DB-SG`).
3. **Description:** Allow MySQL connection from App instances only.
4. **VPC:** Chọn `<VPC_NAME>`.
5. **Inbound rules (Luật vào):**
   * Type: Chọn `MYSQL/Aurora` (Cổng 3306) | Source: Chọn **Custom**, click vào ô tìm kiếm và chọn nhóm bảo mật `<SG_ECS_APP_NAME>`.
6. Click **Create**.

---

## 💾 BƯỚC 3: KHỞI TẠO CƠ SỞ DỮ LIỆU RDS MYSQL

### 1. Tạo DB Subnet Group
1. Vào dịch vụ **RDS** ──> Chọn **Subnet groups** ──> Click **Create DB subnet group**.
2. **Name:** Gõ `<DB_SUBNET_GROUP_NAME>` (Ví dụ: `DB-Subnet-Group`).
3. **Description:** Subnet group for Database.
4. **VPC:** Chọn `<VPC_NAME>`.
5. **Add subnets:**
   * Chọn Availability Zones: `<AZ_1>` và `<AZ_2>`.
   * Chọn Subnet ứng với DB Layer: Tích chọn subnet `<PRIVATE_DB_SUBNET_1_NAME>` và `<PRIVATE_DB_SUBNET_2_NAME>`.
6. Click **Create**.

### 2. Khởi tạo Database MySQL
1. Chọn **Databases** ──> Click **Create database**.
2. Cấu hình thiết lập:
   * **Database creation method:** Chọn **Standard create**.
   * **Engine options:** Chọn **MySQL**.
   * **Templates:** Chọn **Dev/Test** hoặc **Production** tùy nhu cầu lab.
   * **Settings:**
     * **DB instance identifier:** `<DB_IDENTIFIER>` (Ví dụ: `database-lab`).
     * **Master username:** `<DB_MASTER_USERNAME>` (Ví dụ: `admin` hoặc `root`).
     * **Master password:** Nhập mật khẩu bảo mật của bạn.
   * **Connectivity:**
     * **Virtual private cloud (VPC):** Chọn `<VPC_NAME>`.
     * **DB Subnet Group:** Chọn `<DB_SUBNET_GROUP_NAME>`.
     * **Public access:** Chọn **No** (Chặn hoàn toàn từ Internet).
     * **VPC security group:** Chọn **Choose existing** ──> Xóa default group, tích chọn nhóm `<SG_RDS_DB_NAME>`.
     * **Availability Zone:** Chọn `<AZ_1>`.
3. Click **Create database** (Quá trình này mất khoảng 5 - 10 phút).
4. Khi database khởi chạy xong, copy lại **Endpoint** của DB để cấu hình biến môi trường sau này.

---

## ⚖️ BƯỚC 4: THIẾT LẬP APPLICATION LOAD BALANCER (ALB)

### 1. Tạo Target Group
1. Vào dịch vụ **EC2** ──> Chọn **Target Groups** bên sidebar ──> Click **Create target group**.
2. **Target type:** Chọn **IP addresses** (Bắt buộc đối với ECS Fargate vì mỗi container chạy Fargate sẽ nhận một IP động trong Subnet).
3. **Target group name:** `<TARGET_GROUP_NAME>` (Ví dụ: `tg-items`).
4. **Protocol & Port:** Chọn `HTTP` | Cổng `<CONTAINER_PORT>` (Ví dụ: `3000`).
5. **VPC:** Chọn `<VPC_NAME>`.
6. **Health checks (Khám sức khỏe):**
   * **Health check path:** Gõ một API endpoint luôn trả về status 200 OK (Ví dụ: `/health` hoặc `/api/v1/health`).
7. Click **Next** ──> Để trống phần Register targets ──> Click **Create target group**.

### 2. Tạo Application Load Balancer (ALB)
1. Chọn **Load Balancers** bên sidebar ──> Click **Create load balancer** ──> Chọn **Application Load Balancer**.
2. **Load balancer name:** `<ALB_NAME>` (Ví dụ: `domain-alb`).
3. **Scheme:** Chọn **Internet-facing** (Đón nhận lưu lượng công cộng).
4. **IP address type:** Chọn **IPv4**.
5. **Network mapping:**
   * **VPC:** Chọn `<VPC_NAME>`.
   * **Mappings:** Chọn cả 2 AZ (`<AZ_1>` và `<AZ_2>`). Chọn đúng 2 public subnets: `<PUBLIC_SUBNET_1_NAME>` và `<PUBLIC_SUBNET_2_NAME>`.
6. **Security groups:** Xóa default group, chọn nhóm `<SG_ALB_NAME>`.
7. **Listeners and routing:**
   * **Protocol:** `HTTP` | **Port:** `80`.
   * **Default action:** Chọn **Return fixed response**.
   * **Response code:** `404`.
   * **Response body:** Gõ `Not Found` (Trick bảo mật chặn các quét dạo).
8. Click **Create load balancer**.

### 3. Cấu hình Rule điều hướng của ALB
1. Quay lại danh sách Load Balancers, chọn `<ALB_NAME>`.
2. Chuyển sang tab **Listeners and rules** ──> Chọn Listener cổng `HTTP:80` ──> Click **Manage rules** ──> Chọn **Add rule**.
3. **Name / Priority:** Điền số thứ tự ưu tiên (Ví dụ: `1`).
4. **Add condition:** Chọn **Path** ──> Gõ đường dẫn API của bạn: `<API_PATH_PATTERN>` (Ví dụ: `/api/*` hoặc `/api/v1/items/*`).
5. **Add action:** Chọn **Forward to target groups** ──> Chọn target group `<TARGET_GROUP_NAME>`.
6. Click **Save / Create rule**.

---

## 🚢 BƯỚC 5: TRIỂN KHAI ỨNG DỤNG LÊN AWS ECS FARGATE

### 1. Tạo ECS Cluster
1. Vào dịch vụ **Elastic Container Service (ECS)** ──> Chọn **Clusters** ──> Click **Create cluster**.
2. **Cluster name:** `<ECS_CLUSTER_NAME>` (Ví dụ: `Lab-Cluster`).
3. **Infrastructure:** Mặc định chọn **AWS Fargate** (Serverless container).
4. Click **Create**.

### 2. Tạo Task Definition (Định nghĩa Container)
1. Ở menu bên trái, chọn **Task definitions** ──> Click **Create new task definition**.
2. **Task definition family:** Gõ tên định nghĩa (Ví dụ: `<ECS_TASK_FAMILY_NAME>`).
3. **Infrastructure:**
   * **Launch type:** Chọn **AWS Fargate**.
   * **Task size:** Chọn thông số CPU & Memory (Ví dụ: `0.5 vCPU` và `1 GB`).
   * **Task role:** Chọn `<ECS_TASK_ROLE_NAME>` (Cấp quyền cho code kết nối S3).
   * **Task execution role:** Chọn `<ECS_TASK_EXECUTION_ROLE_NAME>` (Cấp quyền cho ECS Agent kéo image/mở Secrets).
4. **Container settings:**
   * **Name:** `<CONTAINER_NAME>`.
   * **Image URI:** Điền link image trên Docker Hub hoặc Amazon ECR (Ví dụ: `<DOCKER_HUB_USERNAME>/<DOCKER_REPOSITORY_NAME>:<TAG>`).
   * **Port mappings:**
     * **Container port:** `<CONTAINER_PORT>` (Ví dụ: `3000`).
     * **Protocol:** `TCP` | **App protocol:** `HTTP`.
5. **Environment variables (Biến môi trường):**
   Cuộn xuống mục **Environment** ──> Cấu hình các biến môi trường cho container:
   * **Bơm trực tiếp (Key-Value):** Điền các biến thông thường như `PORT`, `NODE_ENV`, vv.
   * **Bơm từ Secrets Manager (Dữ liệu bảo mật):**
     * Chọn source từ Secrets Manager ──> Dán chuỗi ARN của secret kèm key tương ứng (Ví dụ: ARN của Database password).
6. Click **Create** để hoàn tất định nghĩa.

### 3. Khởi tạo Service chạy Task trong Cluster
1. Quay lại Cluster `<ECS_CLUSTER_NAME>` vừa tạo ──> Tab **Services** ──> Click **Create**.
2. **Deployment configuration:**
   * **Application type:** Chọn **Service**.
   * **Family:** Chọn `<ECS_TASK_FAMILY_NAME>` | Revision: Chọn bản mới nhất.
   * **Service name:** `<ECS_SERVICE_NAME>`.
   * **Desired tasks:** `2` (Chạy 2 container song song để cân bằng tải và dự phòng).
3. **Networking:**
   * **VPC:** Chọn `<VPC_NAME>`.
   * **Subnets:** Tích chọn 2 private app subnets: `<PRIVATE_APP_SUBNET_1_NAME>` và `<PRIVATE_APP_SUBNET_2_NAME>`.
   * **Security group:** Chọn **Choose existing** ──> Chọn `<SG_ECS_APP_NAME>`.
   * **Public IP:** Chọn **Disabled** (Bảo mật: Container chạy trong vùng private không cần IP Public).
4. **Load balancing:**
   * **Load balancer type:** Chọn **Application Load Balancer**.
   * **Load balancer:** Chọn `<ALB_NAME>`.
   * **Container to load balance:** Chọn `<CONTAINER_NAME>:<CONTAINER_PORT>` ──> Click **Add to load balancer**.
   * **Listener:** Chọn port `80` (hoặc listener cổng bạn cấu hình).
   * **Target group:** Chọn `<TARGET_GROUP_NAME>`.
5. Click **Create** và đợi các task khởi động chuyển sang trạng thái **Running**.

---

## 🌐 BƯỚC 6: TRIỂN KHAI FRONTEND (S3 & CLOUDFRONT)

### 1. Tạo các S3 Buckets
Chúng ta cần tạo 2 Buckets lưu trữ riêng biệt cho Frontend và tài nguyên Media.
1. **S3 Lưu trữ Frontend:**
   * Tạo bucket `<FE_S3_BUCKET_NAME>` (Ví dụ: `frontend-web-ecom-lab`), tắt tùy chọn **Block all public access** để người dùng có thể đọc file.
   * Vào tab **Properties** ──> Bật **Static website hosting** ──> Điền `index.html` cho cả Index và Error document.
   * Vào tab **Permissions** ──> Thêm **Bucket policy** để cho phép đọc file (thay bằng tên bucket thực tế):
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<FE_S3_BUCKET_NAME>/*"
        }
    ]
}
```
2. **S3 Lưu trữ Media (Ảnh sản phẩm/User):**
   * Tạo bucket `<MEDIA_S3_BUCKET_NAME>` (Ví dụ: `media-storage-mini-e`).
   * Bật **ACLs enabled** để Backend có quyền cấp phép hiển thị riêng cho từng ảnh sản phẩm khi upload.

### 2. Thiết lập Tường lửa Ứng dụng Web (WAF)
1. Vào dịch vụ **WAF & Shield** ──> Chọn **Web ACLs** ──> Click **Create web ACL**.
2. **Resource type:** Chọn **CloudFront distributions**.
3. **Associated AWS resources:** Click Add, chọn CloudFront Distribution của bạn (hoặc add sau khi tạo xong CloudFront).
4. **Add rules:** Chọn **Add managed rule groups** của AWS để lọc các traffic bẩn:
   * `Amazon IP Reputation list` (Chặn IP danh tiếng xấu).
   * `Core rule set` (Chặn các lỗ hổng OWASP phổ biến).
   * `SQL database active system` (Chặn tấn công SQL Injection).
5. Hoàn tất các bước tạo.

### 3. Thiết lập CloudFront Distribution (Mạng phân phối nội dung)
1. Vào dịch vụ **CloudFront** ──> Click **Create distribution**.
2. **Origin:**
   * **Origin domain:** Dán link **Bucket website endpoint** từ S3 Frontend (Dạng: `<FE_S3_BUCKET_NAME>.s3-website-<AWS_REGION>.amazonaws.com`).
3. **Web Application Firewall (WAF):**
   * Chọn **Associate web ACL** và chọn đúng Web ACL vừa tạo ở mục 2.
4. **Default cache behavior:**
   * **Viewer protocol policy:** Chọn **Redirect HTTP to HTTPS** để bắt buộc mã hóa toàn bộ dữ liệu.
5. Click **Create distribution** và chờ quá trình phân phối hoàn thành (khoảng 3 - 5 phút). Bạn sẽ nhận được 1 URL phân phối công cộng dạng `https://<CF_ID>.cloudfront.net`.

---

## 🔗 BƯỚC 7: KẾT NỐI FRONTEND & BACKEND (MATCHING FE & BE)

### 1. Phía Backend: Cấu hình CORS
Vì Frontend chạy trên domain CloudFront (hoặc tên miền riêng), còn Backend chạy trên domain Load Balancer (ALB), trình duyệt sẽ chặn request nếu Backend không cho phép.
1. Cấu hình biến môi trường **CORS_ORIGINS** cho Backend của bạn.
2. Điền domain của CloudFront (Ví dụ: `https://<CF_ID>.cloudfront.net`) hoặc tên miền riêng của bạn.
3. *Lưu ý:* Tuyệt đối không để giá trị `*` ở môi trường Production.

### 2. Phía Frontend: Cấu hình API Base URL
1. Tìm file cấu hình môi trường Frontend (Ví dụ: `.env.production` hoặc `src/config.js`).
2. Điền API Base URL trỏ về DNS Name của Load Balancer (ALB):
   * `VITE_API_URL` hoặc `REACT_APP_API_URL` = `http://<ALB_DNS_NAME>`
3. Build mã nguồn Frontend và upload toàn bộ thư mục output (`dist` hoặc `build`) lên S3 Bucket `<FE_S3_BUCKET_NAME>`.

### 3. Phía CloudFront: Ghép nối API về cùng một Domain duy nhất (Tùy chọn)
Để tránh lỗi CORS và tăng tính bảo mật, bạn có thể định tuyến request API đi qua chính CloudFront:
1. Vào CloudFront Distribution ──> Tab **Origins** ──> Click **Create origin**.
   * **Origin domain:** Chọn DNS của Load Balancer (ALB).
   * **Protocol:** Chọn **HTTP only** (Do ALB của chúng ta đang cấu hình lắng nghe cổng 80).
2. Sang tab **Behaviors** ──> Click **Create behavior**.
   * **Path pattern:** Gõ `/api/*`.
   * **Origin:** Chọn ALB vừa add ở trên.
   * **Viewer protocol policy:** Chọn **Redirect HTTP to HTTPS**.
   * **Cache policy:** Chọn **CachingDisabled** (API không nên cache dữ liệu động).
3. Click **Create behavior**. Giờ đây, Frontend có thể gọi API Backend bằng đường dẫn tương đối (Ví dụ: `/api/v1/...`) thông qua cùng một domain của CloudFront.

---

> [!IMPORTANT]
> 🎉 **Hệ thống đã triển khai hoàn tất!**
> Luồng dữ liệu hoạt động khép kín và an toàn tuyệt đối:
> * Người dùng truy cập Frontend qua HTTPS của CloudFront ──> Load file tĩnh từ S3 Frontend.
> * Frontend gọi API ──> Đi qua CloudFront WAF ──> Forward đến Load Balancer (ALB) ──> Đẩy vào 2 Container ECS Fargate ở Private Subnet ──> Query dữ liệu an toàn từ RDS Database ở phân khu cô lập hoàn toàn.
