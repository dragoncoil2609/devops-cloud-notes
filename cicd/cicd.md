# CI/CD với GitHub Actions cho Newbies

Tài liệu giới thiệu và hướng dẫn tự học CI/CD sử dụng GitHub Actions dành cho người mới bắt đầu (newbies).

## 🚀 Giới thiệu Series
Đây là series bài viết từ Hoàng ở **KKloud Tarus**, hướng dẫn từng bước từ lý thuyết cơ bản đến thực hành triển khai thực tế một pipeline CI/CD hoàn chỉnh.

### 🔗 Link bài viết gốc
Bạn có thể đọc toàn bộ series bài viết tại đây:
👉 **[CI/CD Với GitHub Actions Cho Newbie | KKloud Tarus](https://kkloudtarus.net/blog/series/cicd-github-actions-for-newbies)**

---

## 📚 Nội dung chính của Series

### [Phần 1: Khái Niệm CI/CD Và Pipeline Đầu Tay Trên AWS EC2](https://kkloudtarus.net/blog/cicd-concepts-and-first-pipeline-on-aws-ec2)
* **Khái niệm cơ bản:** Phân biệt rõ ràng giữa CI (Continuous Integration), CD (Continuous Delivery) và CD (Continuous Deployment).
* **Các giai đoạn chuẩn của Pipeline:** Đi qua các stage tiêu chuẩn: `Source -> Build -> Test -> Quality Gate -> Package -> Deploy -> Verify`.
* **Thực hành thực tế:** Triển khai một ứng dụng web React + Node.js lên AWS EC2 bằng Docker và thiết lập tự động hóa qua GitHub Actions.
* **Chiến lược Deploy:** Giới thiệu các chiến lược triển khai phổ biến trong thực tế.

### [Phần 2: Những Chỗ Tutorial GitHub Actions Hay Bỏ Qua](https://kkloudtarus.net/blog/things-github-actions-tutorials-tend-to-skip)
Đi sâu vào 9 chủ đề nâng cao và thực tế mà các bài hướng dẫn cơ bản thường bỏ qua:
1. **Concurrency Control:** Kiểm soát số lượng workflow chạy song song (kèm các cạm bẫy với `github.ref`).
2. **Quy tắc đọc file YAML của nhánh:** Cách GitHub Actions kích hoạt event ngoại cảnh.
3. **Họ event `workflow_*`:** Tìm hiểu chi tiết về `workflow_dispatch`, `workflow_call`, `workflow_run` (kèm lưu ý về `head_sha`).
4. **Caching Dependencies:** Tối ưu hóa thời gian build bằng cách cache dependencies.
5. **Matrix Strategy:** Chạy kiểm thử song song trên nhiều môi trường/phiên bản khác nhau.
6. **Docker Hub:** Đẩy ảnh Docker lên Docker Hub thay vì build trực tiếp trên server để giảm tải.
7. **Permissions cho GITHUB_TOKEN:** Cấu hình bảo mật đặc quyền tối thiểu.
8. **OIDC (OpenID Connect) cho AWS:** Loại bỏ hoàn toàn việc sử dụng AWS Access Keys dài hạn và thay bằng OIDC/IAM Role.
9. **Environment & Approval Gate:** Thiết lập các môi trường (Staging, Production) kèm bước phê duyệt thủ công (Required Reviewers).

---

> [!TIP]
> Hãy đọc kỹ Phần 1 để nắm chắc các khái niệm nền tảng trước khi chuyển sang Phần 2 để tối ưu hóa pipeline của bạn đạt chuẩn production.
