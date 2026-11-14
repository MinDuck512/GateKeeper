# PRODUCT REQUIREMENTS DOCUMENT (PRD)
**Dự án:** GateKeeper AI – Agent Quản lý Ra vào, Đăng ký Khách & Thẻ Cư dân

---

## 1. Mục tiêu & Phạm vi Sản phẩm (Scope)

### 1.1. Giai đoạn Cơ bản (MVP)
- Hệ thống đăng nhập phân quyền (Cư dân, Bảo an, Ban Quản lý) qua Supabase Auth, triển khai hoàn chỉnh trên Vercel & Railway.
- AI Agent hỗ trợ cư dân đăng ký khách bằng ngôn ngữ tự nhiên (văn bản/hình ảnh), tự động sinh mã QR/OTP có thời hạn.
- Giao diện chốt bảo an hỗ trợ quét/nhập mã QR/OTP để xác nhận ra vào và ghi log thời gian thực vào PostgreSQL.

### 1.2. Giai đoạn Nâng cao (Advanced)
- **Phát hiện bất thường & HITL:** Agent phát hiện khách lạ tần suất cao, đăng ký ngoài giờ (22:00 – 06:00) hoặc lưu trú qua đêm để kích hoạt luồng HITL chờ Bảo an duyệt.
- **Deterministic Engine:** Module code xác định kiểm chứng hóa đơn điện/nước bậc thang và thuật toán phân bổ tài nguyên/tiện ích công bằng theo căn hộ.
- **Dashboard An ninh & Evaluation:** Phân tích lưu lượng ra vào theo khung giờ, quản lý phản ánh sự cố khép kín và đo lường tỉ lệ cảnh báo đúng của Agent.

---

## 2. Đặc tả Vai trò & Kịch bản Người dùng (User Stories)

| ID | Vai trò | Tính năng | Mô tả User Story | Tiêu chí Chấp nhận (Acceptance Criteria) |
| :--- | :--- | :--- | :--- | :--- |
| **US01** | Cư dân | Đăng ký khách thường | Là cư dân, tôi muốn nhắn tin tự nhiên (VD: *"Cho anh Nam giao hàng SĐT 0912xxx vào sảnh B lúc 14h hôm nay"*) để nhận ngay mã QR/OTP gửi cho khách. | Agent trích xuất đúng Tên, Loại khách, SĐT, Khung giờ; sinh mã QR/OTP hiệu lực trong khung giờ đăng ký. |
| **US02** | Cư dân | Đăng ký khách qua đêm (HITL) | Là cư dân, tôi muốn đăng ký cho người thân ở lại qua đêm kèm ảnh CCCD để bảo an nắm thông tin và phê duyệt. | Agent trích xuất OCR từ ảnh CCCD, nhận diện yêu cầu "qua đêm", chuyển trạng thái sang `PENDING_APPROVAL` và thông báo chờ Bảo an duyệt. |
| **US03** | Cư dân | Kiểm chứng hóa đơn & Đặt tiện ích | Là cư dân, tôi muốn gửi ảnh hóa đơn điện nước hoặc đặt lịch sân cầu lông/BBQ để hệ thống kiểm tra chi phí và hạn mức. | Agent trích xuất số chữ điện/nước, gọi hàm Python tính đúng biểu phí bậc thang tới từng đồng; kiểm tra quota căn hộ trước khi xác nhận đặt chỗ. |
| **US04** | Bảo an | Quét mã & Duyệt HITL | Là bảo an tại cổng, tôi muốn quét mã QR/OTP để cho khách qua cổng và duyệt các yêu cầu khách qua đêm/ngoài giờ ngay trên màn hình trực. | Thời gian tra cứu mã < 500ms; hiển thị rõ thông tin khách, căn hộ mời và nút `Approve` / `Reject` cho các yêu cầu HITL. |
| **US05** | Bảo an / BQL | Cảnh báo bất thường | Là bảo an/BQL, tôi muốn nhận cảnh báo đỏ khi một cá nhân/biển số xe ra vào quá 5 lần/ngày tại nhiều căn hộ khác nhau hoặc đi ngoài giờ. | Node `alert` tự động gắn cờ `ANOMALY_DETECTED`, đẩy thông báo thời gian thực lên màn hình bảo an và Dashboard BQL. |
| **US06** | Ban Quản lý | Dashboard & Eval | Là BQL, tôi muốn xem biểu đồ lưu lượng theo giờ, danh sách phản ánh sự cố và tỉ lệ cảnh báo đúng của AI để tối ưu nhân sự trực cổng. | Hiển thị biểu đồ lưu lượng theo chuẩn GTFS mô phỏng, danh sách ticket phân loại theo mẫu NYC 311 và bảng chỉ số Precision/Recall của Agent. |

---

## 3. Kiến trúc Luồng AI Agent (LangGraph Workflow)

### 3.1. Định nghĩa Trạng thái (AgentState)
```python
from typing import TypedDict, Optional, List, Literal

class GateKeeperState(TypedDict):
    tenant_id: str                  # Mã căn hộ (đảm bảo Tenant Isolation)
    user_id: str                    # ID người gửi yêu cầu
    intent: Literal["visitor_reg", "resident_card", "bill_verify", "incident_report", "facility_booking"]
    raw_input: str                  # Tin nhắn văn bản của cư dân
    media_urls: List[str]           # Ảnh CCCD, biển số xe, hóa đơn hoặc video sự cố
    visitor_info: Optional[dict]    # {name, phone, id_number, license_plate, visit_type, start_time, end_time, is_overnight}
    risk_score: float               # Điểm rủi ro bất thường (0.0 -> 1.0)
    anomaly_reasons: List[str]      # Lý do cảnh báo (ngoài giờ, tần suất cao, blacklist)
    requires_hitl: bool             # Cờ bắt buộc Bảo an phê duyệt
    hitl_status: Optional[Literal["PENDING", "APPROVED", "REJECTED"]]
    pass_data: Optional[dict]       # {pass_id, qr_payload, otp_code, expires_at}
    deterministic_result: Optional[dict] # Kết quả tính phí bậc thang / phân bổ tiện ích
    response_message: str           # Phản hồi trả về cho người dùng