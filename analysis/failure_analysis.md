# Failure Analysis — Lab 18: Production RAG

**Họ và tên:** Phan Hoàng Dũng  
**MSSV:** 2A202601348  
**Pipeline:** Production RAG (M1 Chunking + M5 Enrichment + M2 Hybrid Search + M3 Cross-Encoder Reranking + M4 RAGAS)

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ | Đánh giá |
|--------|---------------|------------|---|----------|
| **Faithfulness** | 0.8194 | 0.6861 | -0.1333 | LLM bị ảnh hưởng bởi conflicting context/outdated policies trong top-3 |
| **Answer Relevancy** | 0.8171 | 0.7559 | -0.0612 | Đạt ngưỡng khá (≥ 0.75), câu trả lời tập trung vào câu hỏi |
| **Context Precision** | 0.9250 | 0.9208 | -0.0042 | Rất cao (> 0.92), Cross-Encoder lọc nhiễu chính xác đưa doc đúng lên top |
| **Context Recall** | 0.9250 | 0.8250 | -0.1000 | Giảm do top-k rerank thu hẹp còn top-3, gây thiếu thông tin ở câu hỏi multi-hop |

---

## Bottom-5 Failures

### #1
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Theo chính sách hiện hành (v2.0), mật khẩu phải được thay đổi mỗi 120 ngày. Chính sách cũ yêu cầu 90 ngày nhưng đã bị thay thế.
- **Got:** 90 ngày (hoặc trả lời theo chính sách cũ do retrieval trích nhầm tài liệu v1.0).
- **Worst metric:** `faithfulness` (Score: 0.0)
- **Error Tree:** Output sai (90 ngày thay vì 120 ngày) → Context sai/lỗi thời (chứa cả policy cũ v1.0 và mới v2.0) → Query OK → Reranker ưu tiên chunk v1.0 vì độ tương đồng từ khóa cao.
- **Root cause:** Temporal / Version Conflict: Hệ thống chưa có metadata filter để lọc bỏ các tài liệu đã hết hiệu lực (`is_active: false` hoặc `version < 2.0`).
- **Suggested fix:** Thêm metadata filtering theo `version` / `effective_date`, hoặc enrichment bổ sung cảnh báo phiên bản vào header của chunk.

### #2
- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** KHÔNG. Nhân viên thử việc chưa được hưởng gói bảo hiểm sức khỏe PVI. Chỉ được tham gia bảo hiểm xã hội bắt buộc.
- **Got:** Có, nhân viên được hưởng hạn mức 200.000.000 VNĐ (hallucination do chỉ retrieve được chunk quyền lợi chung).
- **Worst metric:** `faithfulness` (Score: 0.0)
- **Error Tree:** Output sai (Khẳng định được hưởng) → Context thiếu điều khoản ngoại lệ (chỉ có chunk mô tả gói PVI chung 200 triệu) → Query OK → Retrieval miss chunk quy định thử việc.
- **Root cause:** Granularity mismatch: Đoạn văn mô tả quyền lợi bảo hiểm bị chia cắt khỏi phần "Đối tượng áp dụng / Điều khoản loại trừ đối với nhân viên thử việc".
- **Suggested fix:** Áp dụng Parent-Document Retrieval (trả về full Parent chunk khi Child chunk match) để giữ nguyên vẹn các điều khoản loại trừ.

### #3
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Chỉ trả lời được số ngày phép hoặc mức lương, thiếu 1 trong 2 vế.
- **Worst metric:** `context_recall` (Score: 0.0)
- **Error Tree:** Output thiếu thông tin → Context chỉ chứa tài liệu nghỉ phép, thiếu tài liệu thang bảng lương → Query phức tạp gồm 2 ý độc lập → Retrieval đơn lẻ chỉ match tài liệu có điểm cao hơn.
- **Root cause:** Multi-hop / Multi-domain Query: Câu hỏi yêu cầu thông tin từ 2 tài liệu hoàn toàn tách biệt (`leave_policy.md` và `salary_structure.md`). Top-3 rerank chỉ lấy các chunk của 1 tài liệu có điểm cao nhất.
- **Suggested fix:** Phân rã câu hỏi (Query Decomposition / Sub-queries) thành 2 truy vấn riêng biệt rồi gộp kết quả retrieval trước khi rerank.

### #4
- **Question:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?
- **Expected:** Nghỉ 16-30 ngày cần phê duyệt của Giám đốc điều hành (CEO). Lưu ý: nghỉ trên 14 ngày không lương, nhân viên phải tự đóng phần bảo hiểm của mình.
- **Got:** Trưởng phòng / Giám đốc bộ phận phê duyệt.
- **Worst metric:** `faithfulness` (Score: 0.0)
- **Error Tree:** Output sai cấp phê duyệt → Context trả về quy định nghỉ phép có lương chung (< 5 ngày) thay vì bảng phân cấp thẩm quyền nghỉ không lương dài hạn → Query match sai bảng quy định.
- **Root cause:** Bảng phân cấp phê duyệt bị cắt vụn qua các chunk nhỏ, làm mất liên kết giữa số ngày (khoảng 16-30 ngày) và người phê duyệt tương ứng (CEO).
- **Suggested fix:** Structure-Aware Chunking bảo toàn bảng biểu Markdown (Table-aware parser) để giữ nguyên cấu trúc ma trận phân quyền.

### #5
- **Question:** Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT?
- **Expected:** Laptop 30 triệu nằm trong khoảng 5-50 triệu nên cần Giám đốc phòng ban (Director) phê duyệt. Ngoài ra, mua sắm thiết bị CNTT cần có xác nhận cấu hình kỹ thuật từ phòng CNTT trước khi đề xuất. Cần đính kèm ít nhất 3 báo giá vì trên 10 triệu.
- **Got:** Trả lời được cấp phê duyệt (Director) nhưng thiếu yêu cầu xác nhận kỹ thuật từ phòng CNTT và 3 báo giá.
- **Worst metric:** `context_recall` (Score: 0.3333)
- **Error Tree:** Output thiếu điều kiện phụ → Context chỉ retrieve được hạn mức tài chính, thiếu quy trình mua sắm IT chuyên biệt → Reranker ưu tiên chunk về thẩm quyền tài chính.
- **Root cause:** Information scattering: Quy định mua sắm tài sản nằm ở chính sách tài chính, nhưng quy định duyệt cấu hình lại nằm ở chính sách CNTT.
- **Suggested fix:** Contextual Enrichment (HyQA + Context Prepend) bổ sung các từ khóa liên quan đến "quy trình mua sắm thiết bị IT" vào metadata của chunk.

---

## Case Study (cho presentation)

**Question chọn phân tích:**  
> *"Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?"*

**Error Tree walkthrough:**
1. **Output đúng?** $\rightarrow$ KHÔNG. Output chỉ trả lời được 1 vế (ngày phép hoặc lương).
2. **Context đúng?** $\rightarrow$ KHÔNG ĐẦY ĐỦ. Context chỉ chứa chunk từ `leave_policy.md`, hoàn toàn thiếu chunk từ `salary_structure.md`.
3. **Query rewrite OK?** $\rightarrow$ CHƯA TỐI ƯU. Query nguyên bản là một câu hỏi kép (compound question) gửi trực tiếp vào Hybrid Search khiến vector embedding bị kéo lệch về chủ đề có trọng số từ vựng cao hơn.
4. **Fix ở bước:**
   - **Query Transformation:** Thêm bước Query Decomposition / Sub-question generator:
     - Sub-query 1: *"Nhân viên Senior 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm?"*
     - Sub-query 2: *"Mức lương của nhân viên cấp Senior là bao nhiêu?"*
   - **Multi-retrieval Fusion:** Thực hiện truy vấn song song cho cả 2 sub-queries và hợp nhất top-k trước khi đưa vào Cross-Encoder.

**Nếu có thêm 1 giờ, sẽ optimize:**
1. **Multi-Query / Sub-Question Decomposition:** Xử lý triệt để các câu hỏi kép, tăng Context Recall lên $> 0.95$.
2. **Version & Temporal Filtering:** Bổ sung metadata `version` và `status` để loại bỏ tài liệu cũ (như policy 2023/v1.0), tăng Faithfulness lên $> 0.85$.
3. **Parent-Document Retrieval Expansion:** Khi retrieve trúng child chunk (256 chars), expand lấy toàn bộ parent context (2048 chars) truyền cho LLM để không bị sót các điều khoản loại trừ.
