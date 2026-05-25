# KỊCH BẢN TRIỂN KHAI VÀ KHÔI PHỤC HỆ THỐNG (DEPLOYMENT & DISASTER RECOVERY)

> [!NOTE]
> Tài liệu này liệt kê chi tiết các quy trình chuẩn để triển khai ứng dụng, kịch bản đề phòng rủi ro và các bước phục hồi dữ liệu/hệ thống trong trường hợp xảy ra sự cố đột xuất. Tài liệu dành cho đội ngũ phát triển (Dev) hoặc kỹ thuật viên quản trị (System Admin).

---

## 1. KỊCH BẢN TRIỂN KHAI CHUẨN (DEPLOYMENT STANDARD OPERATING PROCEDURE)

Quá trình cập nhật phiên bản mới cho Website Khách Sạn Bella Phú Quốc bao gồm 3 phân đoạn chính:

### Giai đoạn 1: Chuẩn bị trước khi triển khai (Pre-deployment)
- **Kiểm tra mã nguồn:** Đảm bảo toàn bộ mã nguồn Frontend (HTML/JS/CSS) đã được test đầy đủ ở môi trường Local/Dev `npm run dev`.
- **Review CSDL:** Đảm bảo mô hình dữ liệu (schema) trong file `db.json` không bị lỗi cú pháp JSON và được Map đúng ở file `src/main.js`.
- **Sao lưu:** (Xem mục 2) Backup toàn bộ dữ liệu máy chủ đang chạy trước khi upload Code đè lên.

### Giai đoạn 2: Tiến hành triển khai (Deployment)
1. Trên máy dev (Local), tiến hành build đóng gói phân vùng Frontend bằng Vite:
   ```bash
   npm run build
   ```
2. Tải toàn bộ thư mục `dist/` (chứa các file tĩnh đã nén) lên Web Server/Hosting (VD: Vercel, Netlify, cPanel, Nginx Server, IIS).
3. Nếu nâng cấp tính năng API, tiến hành cập nhật ứng dụng `json-server` trên máy ảo (Cloud VPS) tương ứng. (Start lại PM2 nếu cần `pm2 restart json-server`).

### Giai đoạn 3: Rà soát & Khởi động lại (Post-deployment)
- Xóa bộ nhớ đệm (Clear cache) của CDN hoặc Web Server.
- Truy cập vào trang chủ ẩn danh (Incognito) để kiểm tra giao diện có hoạt động không.
- Gửi thử 1 Form Đặt phòng/Tư vấn để xác nhận đường truyền kết nối đến Backend Database nội bộ (chạy ở cổng `:3000`) vẫn đang hoạt động thông suốt.

---

## 2. KẾ HOẠCH BẢO VỆ VÀ SAO LƯU (BACKUP STRATEGY)

Hệ thống Website sử dụng tệp `db.json` làm cơ sở dữ liệu lưu trữ trực tiếp. Bất cứ khi nào khách hàng tạo đơn tư vấn hoặc Admin đổi giá vé, sự thay đổi sẽ tác động vật lý lên chính file này. Giữ an toàn tệp `db.json` là cốt lõi của việc Backup.

- **Kịch bản sao lưu thủ công (Daily/Weekly):** Admin/Dev truy cập vào VPS / Hosting và thực hiện Copy + Đổi tên file `db.json` sang chuỗi theo ngày như `db_backup_25_05_26.json`. Sau đó có thể tải về lưu trên Google Drive hoặc ổ cứng quản trị.
- **Kịch bản sao lưu tự động (Tương lai):** Kỹ thuật viên có thể bổ sung 1 đoạn Crontab (Linux) hoặc Task Scheduler (Windows) để mỗi ngày (như 2:00 AM sáng) hệ thống tiến hành nén lại thư mục chạy file json-server hiện hữu.

> [!WARNING]
> Không bao giờ để cấp quyền (Chmod/Permission) truy cập Public đối với cấu trúc thư mục chứa file `db.json` hoặc phơi bày cổng API này ngoại trừ việc đã cấu hình CORS bảo mật và chặn kết nối trực tiếp.

---

## 3. CÁC KỊCH BẢN ỨNG PHÓ SỰ CỐ (DISASTER RECOVERY)

### Kịch bản 1: Mất điện diện rộng tại Datacenter hoặc tắt Server đột ngột (Hard Crash)
- **Rủi ro:** Hệ thống Backend có khả năng ngừng hoạt động. Frontend vẫn hiển thị nhưng giá trị phòng sẽ ở trạng thái trống hoặc báo lỗi Console, Form đặt phòng báo Failed to connect.
- **Cách khôi phục:**
  1. Boot lại máy chủ ảo (Nếu bị down).
  2. Ở Terminal, kích hoạt lại hệ thống json-server bằng PM2 để đảm bảo duy trì sau reboot:
     ```bash
     pm2 start npm --name "bella-api" -- run start:server
     pm2 save
     ```
  3. Kiểm tra Logs để biết liệu file `db.json` có bị can thiệp gây lỗi tham số trong lúc tắt máy ảo không: `pm2 logs bella-api`.

### Kịch bản 2: Lỗi ghi đè khiến file `db.json` bị mất, rỗng hoặc sai định dạng JSON (Data Corruption)
- **Rủi ro:** Đây là rủi ro nguy hiểm nhất vì hệ thống sẽ sập hoặc không hiển thị mức giá. Giai đoạn này khách sẽ không thể thực hiện Book phòng.
- **Cách khôi phục:**
  1. Dừng ngay Process JSON-Server nhằm bảo vệ an toàn: `pm2 stop bella-api`.
  2. Lấy bản Backup mới nhất từ Google Drive hoặc folder History Backup trên máy ảo. Dán và ghi đè đổi tên trở lại thành `db.json`.
  3. Sử dụng các trang web như JSONLint để xác nhận tính toàn vẹn format (Format check).
  4. Bật lại JSON-Server `pm2 restart bella-api`. (Thời gian ước tính: 5 - 10 phút).

### Kịch bản 3: Sập Hosting chứa Web tĩnh (Frontend Server Down)
- **Rủi ro:** Không thể truy cập trang chủ khách sạn (Vd: Netlify / Vercel sập / hết băng thông).
- **Cách khôi phục:**
  1. Với kiến trúc tách rời của Vite (Decoupled), dữ liệu của chúng ta vẫn an toàn tại `json-server`.
  2. Kỹ thuật viên ngay lập tức đẩy (Deploy) thư mục `dist/` sang một nền tảng Web Server dự phòng khác (VD: Từ Netlify chuyển nhanh qua Github Pages hoặc Surge.sh).
  3. Update lại DNS của Domain Bella Hotel trỏ về máy chủ tĩnh mới. (Thời gian ước tính mất thêm khoản cập nhật DNS, tầm 10-30 phút là truy cập lại bình thường).

### Kịch bản 4: Trang Quản Trị Khách Sạn (Admin Panel) không đồng bộ giá được
- **Rủi ro:** Do lỗi Cache trên các trình duyệt của Khách Hàng (Tình trạng hay gặp nếu thiết lập sai CDN/Catch). Admin thấy giá đổi nhưng khách không cập nhật do bị dính phiên Cache cũ đối với Frontend.
- **Cách khôi phục:** 
  1. Lệnh cho Hosting/CDN thực thi hành động "Purge Cache".
  2. Trong trường hợp khách hàng báo không thấy cập nhật, hãy hướng dẫn khách làm mới trang bằng `Ctrl + F5` hoặc xóa Cache trên trình duyệt Safari/Chrome ở di động.
  3. **Lưu ý lâu dài:** Bởi vì hệ thống sử dụng Fetch JSON API không định danh, khi viết API Endpoint trong `main.js`, nếu cần, có thể thêm một Timestamp Random vào đuôi URL để ép bỏ Cache tự động ở client (Trình độ Dev: Thêm `?v=timestamp`).
