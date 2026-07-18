# 📘 HƯỚNG DẪN TRIỂN KHAI ỨNG DỤNG CƠ BẢN TRÊN LINUX (ON-PREMISE / VPS)

Tài liệu này hướng dẫn chi tiết các bước chuẩn bị hệ thống, cài đặt dịch vụ và triển khai ứng dụng đa lớp (Backend + Frontend) chạy dưới dạng Service trên hệ điều hành Linux (Ubuntu/Debian), kết hợp Nginx và Docker.

---

## 🛠️ PHẦN 1: CÁC LỆNH LINUX CƠ BẢN DÙNG TRONG DEPLOY

Dưới đây là các lệnh Linux thiết yếu mà một DevOps Engine thường xuyên sử dụng để tương tác và vận hành hệ thống:

```bash
# --- Quản lý Thư mục & File ---
pwd                                 # Xem đường dẫn thư mục hiện tại
cd /path/to/folder                  # Di chuyển sang thư mục khác
ls -lta                             # Liệt kê tất cả file/thư mục (sắp xếp từ mới đến cũ)
mkdir -p /parent/child              # Tạo thư mục mới (bao gồm cả thư mục cha nếu chưa có)
touch index.html                    # Tạo file mới trống
rm -rf /path/to/target              # Xóa thư mục/file (Lưu ý: -f là cưỡng chế xóa không hỏi lại)
cp -r folder/ /destination/         # Sao chép thư mục
mv /source /destination             # Di chuyển hoặc đổi tên file/thư mục

# --- Quản lý User & Quyền hạn ---
whoami                              # Kiểm tra user hiện tại đang đăng nhập
cat /etc/passwd                     # Xem toàn bộ user và quyền hạn trên hệ thống
sudo usermod -aG group_name user    # Thêm user vào group (Ví dụ: usermod -aG docker ubuntu)
sudo deluser user group_name        # Xóa user ra khỏi group
sudo groupadd developers            # Tạo một group mới

# --- Quản lý Tiến trình & Log ---
tail -n 100 -f app.log              # Xem 100 dòng log cuối và theo dõi thời gian thực
netstat -tlpun                      # Xem danh sách các port mạng đang lắng nghe (Active Ports)
```

---

## 🛜 PHẦN 2: CẤU HÌNH IP TĨNH (CHO MÔI TRƯỜNG ON-PREMISE)

Để server chạy ổn định và không bị thay đổi IP sau mỗi lần khởi động lại:

1. Chuyển sang quyền root:
   ```bash
   sudo -i
   ```
2. Mở file cấu hình Netplan bằng trình soạn thảo `nano` (ấn `Tab` để tự động hoàn thành tên file):
   ```bash
   nano /etc/netplan/01-netcfg.yaml
   ```
3. Cấu hình nội dung mạng tĩnh:
   ```yaml
   network:
     version: 2
     renderer: networkd
     ethernets:
       <INTERFACE_NAME>: # Thay bằng tên card mạng thật của bạn (Ví dụ: eth0, ens33)
         dhcp4: false
         addresses:
           - <STATIC_IP>/<CIDR_PREFIX> # Ví dụ: 192.168.1.110/24
         nameservers:
           addresses:
             - 8.8.8.8
             - 8.8.4.4
         routes:
           - to: default
             via: <GATEWAY_IP> # Ví dụ: 192.168.1.1
   ```
4. Áp dụng cấu hình:
   ```bash
   netplan apply
   ```

---

## ☕ PHẦN 3: TRIỂN KHAI BACKEND (SYSTEMD SERVICE)

### 1. Chuẩn bị môi trường & Source Code
1. Cài đặt các thư viện mạng và cài Java, Maven:
   ```bash
   apt update
   apt install -y net-tools openjdk-21-jdk openjdk-21-jre maven git
   ```
2. Clone dự án và phân quyền sở hữu thư mục cho user chạy backend:
   ```bash
   # Tạo thư mục và tải code
   mkdir -p <PROJECT_DIR> && cd <PROJECT_DIR>
   git clone <YOUR_BACKEND_GIT_URL>
   
   # Tạo user hệ thống riêng cho backend chạy ngầm (tăng tính bảo mật)
   adduser --system --group <BACKEND_SYSTEM_USER>
   chown -R <BACKEND_SYSTEM_USER>:<BACKEND_SYSTEM_USER> <PROJECT_DIR>
   chmod -R 755 <PROJECT_DIR>
   ```

### 2. Triển khai Database MySQL
1. Cài đặt MySQL Server:
   ```bash
   apt install mysql-server -y
   ```
2. Cấu hình cho phép truy cập DB từ xa (nếu cần):
   ```bash
   nano /etc/mysql/mysql.conf.d/mysqld.cnf
   # Sửa dòng: bind-address = 127.0.0.1 thành 0.0.0.0
   systemctl restart mysql
   ```
3. Tạo Database và User riêng cho dự án:
   ```bash
   mysql -u root
   ```
   ```sql
   CREATE DATABASE IF NOT EXISTS `<DB_NAME>` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   CREATE USER '<DB_USER>'@'%' IDENTIFIED BY '<DB_PASSWORD>';
   GRANT ALL PRIVILEGES ON `<DB_NAME>`.* TO '<DB_USER>'@'%';
   FLUSH PRIVILEGES;
   ```
4. Cập nhật thông số kết nối Database trong file cấu hình (Ví dụ: `application.properties` hoặc `.env`) của code backend tương ứng với thông tin vừa tạo ở trên.

### 3. Build mã nguồn và thiết lập Service
1. Build dự án bỏ qua unit tests:
   ```bash
   mvn clean install -DskipTests=true
   ```
2. Tạo file Service để hệ thống quản lý tự khởi chạy cùng OS:
   ```bash
   nano /etc/systemd/system/<BACKEND_SERVICE_NAME>.service
   ```
   Dán nội dung sau vào file:
   ```ini
   [Unit]
   Description=Backend Application Service
   After=network.target

   [Service]
   Type=simple
   User=<BACKEND_SYSTEM_USER>
   WorkingDirectory=<BACKEND_WORK_DIR>
   ExecStart=/usr/bin/java -jar target/<JAR_FILE_NAME>.jar
   Restart=always
   RestartSec=10
   StandardOutput=append:/var/log/<LOG_DIR_NAME>/backend.log
   StandardError=append:/var/log/<LOG_DIR_NAME>/backend.err

   [Install]
   WantedBy=multi-user.target
   ```
3. Tạo thư mục log và chạy service:
   ```bash
   mkdir -p /var/log/<LOG_DIR_NAME> && chown -R <BACKEND_SYSTEM_USER>:<BACKEND_SYSTEM_USER> /var/log/<LOG_DIR_NAME>
   
   systemctl daemon-reload
   systemctl start <BACKEND_SERVICE_NAME>.service
   systemctl enable <BACKEND_SERVICE_NAME>.service
   systemctl status <BACKEND_SERVICE_NAME>.service
   ```

---

## 🌐 PHẦN 4: TRIỂN KHAI FRONTEND & CẤU HÌNH NGINX

### 1. Cài đặt NodeJS & Build Frontend
1. Cài đặt Node.js phiên bản LTS:
   ```bash
   curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
   apt install -y nodejs
   ```
2. Tải mã nguồn Frontend và tiến hành Build:
   ```bash
   cd <PROJECT_DIR>
   git clone <YOUR_FRONTEND_GIT_URL>
   cd <FRONTEND_DIR_NAME>
   
   # Cài đặt dependencies và build sang file tĩnh (HTML/JS/CSS)
   npm install
   npm run build
   
   # Di chuyển thư mục build vào thư mục phục vụ web của Nginx
   mkdir -p /var/www/html
   mv build /var/www/html/<FRONTEND_DEPLOY_DIR>
   ```

### 2. Cài đặt và Cấu hình Nginx Web Server
Nginx đóng vai trò Web Server phục vụ file Frontend tĩnh, đồng thời làm Reverse Proxy chuyển tiếp các request API `/api/` về Backend.

1. Cài đặt Nginx:
   ```bash
   apt install nginx -y
   ```
2. Tạo file cấu hình virtual host cho domain:
   ```bash
   nano /etc/nginx/conf.d/<PROJECT_CONF_NAME>.conf
   ```
   Nội dung cấu hình:
   ```nginx
   server {
       listen 80;
       server_name <DOMAIN_NAME>; # Thay bằng domain thực tế của bạn (Ví dụ: domain.com www.domain.com)

       # Đường dẫn tới thư mục chứa code Frontend đã build
       root /var/www/html/<FRONTEND_DEPLOY_DIR>;
       index index.html;

       location / {
           try_files $uri /index.html;
       }

       # Điều hướng các request API về cổng của Backend
       location /api/ {
           proxy_pass http://127.0.0.1:<BACKEND_PORT>/api/;
           proxy_http_version 1.1;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```
3. Kiểm tra cú pháp cấu hình Nginx:
   ```bash
   nginx -t
   ```
4. Khởi động lại dịch vụ:
   ```bash
   systemctl restart nginx
   ```

---

## 🐳 PHẦN 5: CONTAINER HÓA VỚI DOCKER & DOCKER COMPOSE

Sử dụng Docker giúp đơn giản hóa việc triển khai, không cần cài cắm các runtime môi trường thủ công trên OS.

### 1. Cài đặt nhanh Docker & Docker Compose
1. Viết script cài đặt: `nano install-docker.sh`
   ```bash
   #!/bin/bash
   set -e
   sudo apt update
   sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt update
   sudo apt install -y docker-ce
   sudo systemctl start docker
   sudo systemctl enable docker
   # Cài đặt docker-compose v1
   sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   sudo chmod +x /usr/local/bin/docker-compose
   docker --version
   docker-compose --version
   ```
2. Cấp quyền và chạy file script:
   ```bash
   chmod +x install-docker.sh
   ./install-docker.sh
   ```

### 2. Triển khai bằng Docker Compose
Tạo file `docker-compose.yml` ở thư mục dự án để gom Backend và Database chạy chung:

```yaml
version: '3.8'

services:
  backend:
    build:
      context: ./<BACKEND_DIR_NAME>
      dockerfile: Dockerfile
    ports:
      - "<HOST_BACKEND_PORT>:<CONTAINER_BACKEND_PORT>"
    depends_on:
      - db
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/<DB_NAME>
      SPRING_DATASOURCE_USERNAME: <DB_USER>
      SPRING_DATASOURCE_PASSWORD: <DB_PASSWORD>
    container_name: <BACKEND_CONTAINER_NAME>
    restart: always

  db:
    image: mysql:8.0
    volumes:
      - mysql-db-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: <DB_ROOT_PASSWORD>
      MYSQL_DATABASE: <DB_NAME>
      MYSQL_USER: <DB_USER>
      MYSQL_PASSWORD: <DB_PASSWORD>
    ports:
      - "<HOST_DB_PORT>:3306"
    container_name: <DB_CONTAINER_NAME>
    restart: always

volumes:
  mysql-db-data:
```

* **Khởi chạy hệ thống ngầm:**
  ```bash
  docker-compose up -d
  ```
* **Tắt hệ thống container:**
  ```bash
  docker-compose down
  ```
