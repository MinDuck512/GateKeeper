# 1-PAGE BRIEF: GATEKEEPER AI
**Đề tài:** Agent Quản lý Ra vào, Đăng ký Khách & Thẻ Cư dân thông minh cho Khu đô thị

## 1. Bối cảnh & Thực trạng (Problem Statement)
Trong vận hành các khu đô thị quy mô lớn (hàng chục nghìn hộ dân), quy trình kiểm soát an ninh và hỗ trợ cư dân đang gặp 3 điểm nghẽn lớn:
- **Ùn tắc tại chốt bảo an:** Cư dân đăng ký khách đến chơi, shipper, người giúp việc qua nhiều bước ghi chép thủ công, gây ùn tắc cổng vào giờ cao điểm và khó kiểm soát người lạ.
- **Quá tải yêu cầu dịch vụ & Tranh chấp phí:** Các phản ánh sự cố (kèm ảnh/video), yêu cầu cấp thẻ cư dân và khiếu nại hóa đơn điện/nước dồn dập vào giờ cao điểm với nhiều trường hợp ngoại lệ, vượt quá khả năng xử lý thủ công của Ban Quản lý (BQL).
- **Rủi ro an ninh & Sai sót tính toán:** Bảo an khó phát hiện khách lạ ra vào với tần suất bất thường; việc giải thích biểu phí bậc thang hoặc phân bổ tiện ích nội khu dễ xảy ra nhầm lẫn.

## 2. Giải pháp Đề xuất (Proposed Solution)
**GateKeeper AI** là hệ thống AI Agent đa phương thức kết hợp cùng bộ máy tính toán xác định (Deterministic Engine), tạo ra quy trình xử lý khép kín (end-to-end) từ Cư dân -> Chốt Bảo an -> Ban Quản lý:
- **Đăng ký khách & Cấp thẻ bằng ngôn ngữ tự nhiên:** Cư dân nhắn tin hoặc gửi ảnh CCCD/biển số xe; Agent tự động bóc tách thông tin, xác minh và sinh mã QR/OTP ra vào theo thời gian thực.
- **Giám sát bất thường & Duyệt HITL (Human-in-the-Loop):** Tự động phát hiện khách lạ tần suất cao, khách đi ngoài khung giờ hoặc lưu trú qua đêm để chuyển sang trạng thái chờ Bảo an phê duyệt trực tiếp.
- **Xử lý đa phương thức & Kiểm chứng quy định bằng Code xác định:** Tiếp nhận ảnh hóa đơn/sự cố, dùng LLM trích xuất dữ liệu và gọi hàm code xác định để kiểm chứng phí điện nước bậc thang hoặc phân bổ hạn mức tiện ích công bằng.

## 3. Đối tượng Người dùng (Target Users)
1. **Cư dân (App Mobile/Web):** Đăng ký khách nhanh, nhận mã QR/OTP, đăng ký thẻ cư dân, đặt tiện ích và gửi yêu cầu kiểm tra hóa đơn/phản ánh sự cố.
2. **Bảo an (Màn hình Chốt cổng):** Quét mã QR/OTP thời gian thực, nhận cảnh báo an ninh và phê duyệt (HITL) các trường hợp khách qua đêm/ngoài giờ.
3. **Ban Quản lý (Web Dashboard):** Giám sát lưu lượng ra vào theo khung giờ, xem thống kê cảnh báo, xử lý yêu cầu dịch vụ và theo dõi chỉ số đánh giá (Eval) của AI Agent.

## 4. Kiến trúc Kỹ thuật & Ràng buộc Cốt lõi (Tech Stack & Constraints)
- **Tech Stack:** GPT-4o-mini (LLM đa phương thức), LangGraph (luồng Agent: `register` → `verify` → `issue_pass` → `alert`), FastAPI (Backend), Next.js (Frontend), PostgreSQL + Supabase Auth (Database & Phân quyền), triển khai trên Railway & Vercel.
- **Ràng buộc An toàn & Bảo mật:**
  - **Cách ly dữ liệu (Tenant Isolation):** Áp dụng Row-Level Security (RLS) đảm bảo cư dân căn hộ nào chỉ truy cập được lịch sử ra vào và hóa đơn của căn hộ đó.
  - **Code xác định (Deterministic Logic):** Tuyệt đối không dùng LLM để nhẩm tính toán học; mọi phép tính phí bậc thang và kiểm tra hạn mức tiện ích phải thực thi bằng hàm Python độc lập.
  - **Hiệu năng thời gian thực:** Phản hồi quét QR tại chốt cổng dưới 500ms.

## 5. Thước đo Thành công & Dữ liệu Tham chiếu (Success Metrics & Benchmarks)
- **Hiệu quả vận hành:** Giảm thời gian đăng ký & qua cổng từ ~3 phút xuống < 15 giây; đạt độ chính xác 100% trong tính toán kiểm chứng biểu phí.
- **Dữ liệu kiểm thử tham chiếu:** Mô phỏng tải bằng bộ dữ liệu NYC 311 Service Requests (yêu cầu dịch vụ), Building Data Genome Project 2 / UK-DALE (tiêu thụ năng lượng) và chuẩn GTFS (luồng di chuyển).