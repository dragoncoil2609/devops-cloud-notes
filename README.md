# ☁️ DevOps & Cloud Notes

Chào mừng bạn đến với kho lưu trữ ghi chép và hướng dẫn thực hành DevOps & Cloud của tôi. Đây là nơi tôi tổng hợp kiến thức, các bài Lab thực tế và tài liệu triển khai hệ thống (Deployment) trên môi trường Cloud (AWS) cũng như On-Premise/VPS. 

Repository này được xây dựng và thiết kế như một **Portfolio dự án thực tế** để trình bày năng lực thiết kế kiến trúc hệ thống, tự động hóa CI/CD, container hóa và bảo mật hạ tầng mạng.

---

## 🛠️ Tech Stack & Kỹ Năng Đang Ghi Chép / Thực Hành

<p align="left">
  <img src="https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white" alt="AWS" />
  <img src="https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white" alt="Docker" />
  <img src="https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white" alt="Kubernetes" />
  <img src="https://img.shields.io/badge/github%20actions-%232088FF.svg?style=for-the-badge&logo=github-actions&logoColor=white" alt="GitHub Actions" />
  <img src="https://img.shields.io/badge/linux-%23FCC624.svg?style=for-the-badge&logo=linux&logoColor=black" alt="Linux" />
  <img src="https://img.shields.io/badge/nginx-%23009639.svg?style=for-the-badge&logo=nginx&logoColor=white" alt="Nginx" />
</p>

* **Cloud Provider (AWS):** VPC, EC2, ECS Fargate, Lambda, API Gateway, Application Load Balancer (ALB), RDS MySQL, S3, CloudFront, WAF, IAM, Route 53, OIDC.
* **DevOps & CI/CD:** GitHub Actions (Caching, Concurrency, Matrix, OIDC AWS Integration), GitLab Runner, Docker, Docker Compose, Harbor Registry.
* **Hệ thống & Mạng:** Linux Administration (Ubuntu/Debian, systemd services, netplan), Cấu hình tên miền (Domain) và định tuyến proxy Nginx.

---

## 📂 Bản Đồ Thư Mục & Tài Liệu Triển Khai

Kho lưu trữ được cấu trúc một cách khoa học để dễ dàng tra cứu và học hỏi:

| Thư mục / File | Mô tả Nội dung | Dịch vụ & Công nghệ chính |
| :--- | :--- | :--- |
| 📁 **[Deployment/](file:///d:/LearnDevop/devops-cloud-notes/Deployment/)** | **Thư mục chứa toàn bộ tài liệu cấu hình & triển khai hạ tầng** | |
| ├── 📘 [AWS-Deploy-ec2.md](file:///d:/LearnDevop/devops-cloud-notes/Deployment/AWS-Deploy-ec2.md) | Triển khai Docker App trên EC2 trong Auto Scaling Group sau Load Balancer, kết nối RDS Multi-AZ, SSH an toàn qua EICE (không mở Port 22 public). | VPC, EC2, ALB, ASG, RDS, EICE, Docker |
| ├── 📘 [AWS-Deploy_ecs.md](file:///d:/LearnDevop/devops-cloud-notes/Deployment/AWS-Deploy_ecs.md) | Kiến trúc ứng dụng đa lớp (Multi-tier) trên ECS Fargate Serverless. Routing traffic động và quản lý file tĩnh S3 + CloudFront WAF. | ECS Fargate, ALB, RDS, S3, CloudFront, WAF |
| ├── 📘 [AWS-Deploy-lamda.md](file:///d:/LearnDevop/devops-cloud-notes/Deployment/AWS-Deploy-lamda.md) | Triển khai mô hình Serverless hoàn chỉnh: API Gateway tích hợp AWS Lambda (trong VPC) kết nối RDS và hỗ trợ cả ZIP/Container Image (Docker ECR). | Lambda, API Gateway, ECR, RDS, Serverless |
| ├── 📘 [AWS-Deploy-Frontend-Single-Domain.md](file:///d:/LearnDevop/devops-cloud-notes/Deployment/AWS-Deploy-Frontend-Single-Domain.md) | Cấu hình triển khai Frontend tối ưu hóa bảo mật và tốc độ trên AWS CloudFront. | S3, CloudFront, SSL/TLS, ACM |
| ├── 📘 [Linux-Deploy-Basic.md](file:///d:/LearnDevop/devops-cloud-notes/Deployment/Linux-Deploy-Basic.md) | Cẩm nang triển khai On-Premise/VPS cơ bản: Các lệnh Linux cốt lõi, IP tĩnh Netplan, triển khai Backend/Frontend chạy dưới dạng systemd service, proxy Nginx. | Ubuntu, systemd, Nginx, MySQL, Docker |
| 📁 **[cicd/](file:///d:/LearnDevop/devops-cloud-notes/cicd/)** | **Tự động hóa tích hợp & triển khai liên tục** | |
| └── 📘 [cicd.md](file:///d:/LearnDevop/devops-cloud-notes/cicd/cicd.md) | Lý thuyết cơ bản về CI/CD và định hướng tối ưu hóa pipeline GitHub Actions nâng cao cho môi trường Production. | GitHub Actions, YAML CI/CD |
| 📁 **[k8s/](file:///d:/LearnDevop/devops-cloud-notes/k8s/)** | **Điều phối và quản lý Container quy mô lớn** | |
| └── 📘 [k8s-aws-eks.md](file:///d:/LearnDevop/devops-cloud-notes/k8s/k8s-aws-eks.md) | Ghi chép và cấu hình thực tế trên dịch vụ Kubernetes của Amazon (EKS). | AWS EKS, kubectl, Kubernetes |
| 📁 **[Domain_withGithubSubnet/](file:///d:/LearnDevop/devops-cloud-notes/Domain_withGithubSubnet/)** | **Cấu hình mạng nâng cao** | |
| └── 📘 [doc.md](file:///d:/LearnDevop/devops-cloud-notes/Domain_withGithubSubnet/doc.md) | Hướng dẫn định tuyến IP và liên kết tên miền chuyên sâu. | DNS, Subnet, Route 53 |

---

> [!TIP]
> Tất cả tài liệu trong repo này đã được chuẩn hóa dưới dạng **Template Tổng quát (Generalized Templates)**. Bạn chỉ cần thay thế các biến cấu hình dạng `<PLACEHOLDER>` (như `<VPC_NAME>`, `<RDS_DB_ENDPOINT>`, vv.) là có thể áp dụng ngay cho dự án của mình!
