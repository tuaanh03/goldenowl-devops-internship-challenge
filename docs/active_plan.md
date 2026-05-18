## GIAI ĐOẠN 1: KHẢO SÁT & CHUẨN BỊ MÔI TRƯỜNG LOCAL
**Mục tiêu:** Hiểu ứng dụng và thiết lập nền tảng làm việc ban đầu.

### Khởi tạo dự án:
- Fork repository gốc về tài khoản GitHub cá nhân.
- Clone repository từ tài khoản của bạn về máy tính (Local).

### Thiết lập Repository trên GitHub:
- **Đặt nhánh mặc định (Default Branch):** Chuyển nhánh mặc định từ `main` sang `develop` để các thành viên khi truy cập sẽ thấy nhánh này đầu tiên.
- **Thiết lập luật bảo vệ nhánh (Branch Protection Rules):**
  - **Với nhánh `main`:** Cấu hình yêu cầu phải có Pull Request đã được approve (ít nhất 1 người) và đã pass CI (status check) mới được merge. Chặn push trực tiếp.

### Khảo sát ứng dụng & Kiểm tra bảo mật:
- Kiểm tra mã nguồn, đặc biệt rà soát kỹ file `package.json` để đảm bảo không chứa các dependency lạ hoặc malware.
- Kiểm tra xem ứng dụng có sử dụng biến môi trường (Environment Variables / file `.env`) hay không để chuẩn bị cấu hình cho các bước sau.
- Chạy thử ứng dụng bằng Docker container hóa thay vì cài Node.js trực tiếp trên máy local (ví dụ: mount source code vào container `node:18-alpine` để chạy test).
- Truy cập `localhost:3000` để đảm bảo ứng dụng hoạt động đúng như mô tả.

### Thiết lập Git Flow cơ bản:
- Không làm việc trực tiếp trên nhánh `main`/`master`.
- Tạo nhánh mới để bắt đầu công việc (VD: `git checkout -b feature/dockerize-app`).

---

## GIAI ĐOẠN 2: CONTAINERIZATION (ĐÓNG GÓI ỨNG DỤNG)
**Mục tiêu:** Đưa ứng dụng vào môi trường Docker chuẩn bị cho việc chạy trên Cloud.

### Phân tích yêu cầu đóng gói:
- Xác định môi trường chạy (Node.js version nào).
- Cần tối ưu dung lượng và bảo mật (lên kế hoạch dùng Multi-stage build: 1 stage để test/build, 1 stage để chạy).

### Thực thi:
- Viết `Dockerfile` và `.dockerignore`.
- Build thử image trên máy local (`docker build`).
- Chạy thử container trên máy local (`docker run`) và test lại cổng 3000.

### Commit Code:
- Thực hiện commit đầu tiên với thông điệp rõ ràng: `"chore: add Dockerfile and optimize with multi-stage build"`.

---

## GIAI ĐOẠN 3: CHUẨN BỊ HẠ TẦNG DOCKER HUB VÀ GOOGLE CLOUD (GCP)

### Thiết lập Registry (Docker Hub):
- Tạo một repository trên Docker Hub để lưu trữ Docker images ở chế độ **Public**. (Điều này đảm bảo khi GKE đọc file YAML Deployment, nó có thể pull image thoải mái mà **không cần thiết lập IAM phức tạp** hay cấu hình `imagePullSecrets`).
- Tạo Access Token trên Docker Hub để cấp quyền cho GitHub Actions.

### Thiết lập Kubernetes:
- Tạo cụm GKE (hoặc sử dụng cụm bạn đã có).
- Cài đặt NGINX Ingress Controller trên cụm GKE.
- **Cài đặt GitOps Agent (ví dụ: Argo CD):** Cài đặt Argo CD vào cụm GKE và cấu hình nó để theo dõi repository chứa mã nguồn (hoặc một repository riêng cho manifests).

---

## GIAI ĐOẠN 4: THIẾT KẾ & KIỂM THỬ KUBERNETES MANIFESTS
**Mục tiêu:** Định nghĩa kiến trúc hệ thống trên K8s trước khi tự động hóa.

**Chuyển nhánh làm việc:** Tạo nhánh mới (VD: `git checkout -b feature/k8s-manifests`).

### Thiết kế các Component:
- **Deployment:** Khai báo số lượng Pod (chạy tối thiểu 2 để đảm bảo sẵn sàng cao), khai báo giới hạn tài nguyên. Chỉ định thẳng tên image Public từ Docker Hub.
- **Service:** Khai báo ClusterIP để các Pod giao tiếp nội bộ.
- **Ingress:** Cấu hình luật định tuyến để Nginx Ingress nhận traffic từ ngoài và đẩy vào Service nội bộ.
- **HPA (Auto Scaling):** Cấu hình tự động tăng số lượng Pod khi CPU tăng cao.

### Kiểm thử thủ công (Manual Test):
- Commit các file manifest vào repository.
- Truy cập giao diện của Argo CD để xem nó có tự động phát hiện và đồng bộ (sync) ứng dụng lên GKE thành công hay không.

### Commit Code:
- `"feat: add Kubernetes manifests for deployment, service, ingress, and HPA"`.

---

## GIAI ĐOẠN 5: THIẾT LẬP CI/CD PIPELINE (GITHUB ACTIONS)
**Mục tiêu:** Tự động hóa toàn bộ quy trình.

### Bảo mật thông tin:
- Đưa các thông tin nhạy cảm vào mục **Settings > Secrets and variables > Actions** của GitHub. Với mô hình GitOps, chúng ta chỉ cần:
  - `DOCKERHUB_USERNAME`: Tên đăng nhập Docker Hub.
  - `DOCKERHUB_TOKEN`: Access Token của Docker Hub.

### Xây dựng luồng CI (Continuous Integration):
- Tạo file workflow chạy khi có mã mới đẩy lên các nhánh `feature/*`.
- **Quy trình:** Checkout code -> Cài đặt Node.js -> Chạy `npm test`.
- **Commit Code:** `"ci: add GitHub Action workflow for running tests"`.

### Xây dựng luồng CD (Continuous Deployment):
- Tạo file workflow chạy khi mã được hợp nhất (merge) vào `main`/`master`.
- **Quy trình:** Đăng nhập Docker Hub -> Build & Push Docker Image (gắn tag là Git SHA) -> Checkout code -> Dùng công cụ (sed/yq) để cập nhật tag image mới vào file `deployment.yaml` -> Commit và push thay đổi vào `main`.
- **Commit Code:** `"cd: add automated deployment pipeline to GKE"`.

---

## GIAI ĐOẠN 6: TÍCH HỢP, KIỂM THỬ TOÀN DIỆN & LÀM TÀI LIỆU (QUAN TRỌNG NHẤT)
**Mục tiêu:** Chứng minh kết quả và ghi điểm với nhà tuyển dụng qua tài liệu.

### Kích hoạt Pipeline:
- Tạo Pull Request (PR) từ nhánh feature vào main.
- Quan sát CI chạy test pass.
- Merge PR vào main để kích hoạt CD. Quan sát CD build và deploy lên GKE.

### Kiểm chứng thực tế:
- Truy cập vào public IP/Domain của Ingress xem ứng dụng trả về chuỗi JSON chính xác không.

### Viết Tài liệu (README.md):
- Vẽ một sơ đồ kiến trúc (Flow Diagram) mô tả luồng đi của Code (từ lúc push đến lúc deploy) và luồng đi của Traffic (từ User qua Ingress vào Pod).
- Cập nhật file README: Chèn sơ đồ vào, giải thích ngắn gọn lý do chọn kiến trúc (Tại sao dùng Ingress, cấu hình HPA thế nào). Cung cấp link URL của ứng dụng đã chạy thực tế.

### Commit cuối cùng:
- `"docs: update README with architecture diagram and live URL"`.