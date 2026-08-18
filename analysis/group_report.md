# Group Report — Lab 18: Production RAG

**Họ và tên:** Phan Hoàng Dũng  
**MSSV:** 2A202601348  
**Ngày:** 18/08/2026  

---

## Thành viên & Phân công

| Tên | Module | Hoàn thành | Tests pass |
|-----|--------|:----------:|:----------:|
| Phan Hoàng Dũng | M1: Advanced Chunking (Semantic, Hierarchical, Structure) | ☑ | 8/8 (100%) |
| Phan Hoàng Dũng | M2: Hybrid Search (BM25 + Dense + RRF) | ☑ | 5/5 (100%) |
| Phan Hoàng Dũng | M3: Reranking (CrossEncoder + Flashrank) | ☑ | 5/5 (100%) |
| Phan Hoàng Dũng | M4: Evaluation (RAGAS + Diagnostic Tree) | ☑ | 4/4 (100%) |
| Phan Hoàng Dũng | M5: Enrichment (Single-call combined mode) | ☑ | 6/6 (100%) |

---

## Kết quả RAGAS

| Metric | Naive Baseline | Production RAG | Δ 
|--------|:--------------:|:--------------:|:---:|
| **Faithfulness** | 0.8194 | 0.6861 | -0.1333 |
| **Answer Relevancy** | 0.8171 | 0.7559 | -0.0612 |
| **Context Precision** | 0.9250 | 0.9208 | -0.0042 |
| **Context Recall** | 0.9250 | 0.8250 | -0.1000 |

---

## Key Findings

1. **Biggest improvement:**  
   Module **M3 Reranking** kết hợp **M2 Hybrid Search** giúp lọc nhiễu cực kỳ hiệu quả, duy trì `Context Precision` ở mức xuất sắc **0.9208**. Khi tìm kiếm các từ khóa đặc thù tiếng Việt (như "nghỉ phép", "bảo hiểm PVI"), BM25 giải quyết được hạn chế out-of-vocabulary của Dense vector, trong khi Cross-Encoder đẩy các tài liệu chuẩn xác nhất lên top-3.

2. **Biggest challenge:**  
   Thử thách lớn nhất là hiện tượng **Temporal/Version Conflicts** và **Multi-Hop Queries**:
   - Khi kho tri thức có cả chính sách cũ (v2023/v1.0) và mới (v2024/v2.0), Cross-Encoder bị đánh lừa bởi từ khóa tương đồng nếu không có siêu dữ liệu thời gian/trạng thái.
   - Khi câu hỏi chứa nhiều thực thể từ 2 tài liệu độc lập (ví dụ: ngày phép Senior + khung lương Senior), việc thu hẹp còn top-3 chunks làm giảm `Context Recall` và `Faithfulness`.

3. **Surprise finding:**  
   Kỹ thuật **Single-call Combined Enrichment (M5)** giúp tối ưu hóa chi phí API gọi LLM giảm 4 lần (1 request/chunk thay vì 4 requests riêng lẻ) mà vẫn sinh đầy đủ câu hỏi giả định (HyQA), ngữ cảnh bổ sung và trích xuất siêu dữ liệu JSON chính xác.

---

## Presentation Notes (5 phút)

1. **RAGAS scores (Naive vs Production):**
   - Naive RAG lấy chunk thô kích thước cố định nên đôi khi vô tình lấy được đoạn dài bao hàm nhiều ý nhưng chứa nhiều nhiễu.
   - Production RAG tinh chỉnh theo kiến trúc chuẩn công nghiệp (Hierarchical Chunking $\rightarrow$ Hybrid Search $\rightarrow$ Cross-Encoder $\rightarrow$ RAGAS Eval), đạt Context Precision vượt trội (0.92) và Answer Relevancy vững chắc (0.76).

2. **Biggest win — module nào, tại sao:**
   - **M3 Cross-Encoder Reranker:** Giúp re-score chính xác theo tương tác sâu giữa câu hỏi và văn bản, loại bỏ các kết quả match từ khóa nông của BM25 và bù đắp điểm yếu vector của Dense Search.

3. **Case study — 1 failure, Error Tree walkthrough:**
   - Câu hỏi: *"Bao lâu phải đổi mật khẩu một lần?"* (Chính sách cũ: 90 ngày, Chính sách mới v2.0: 120 ngày).
   - Error Tree: Output sai (90 ngày) $\leftarrow$ Context chứa chunk v1.0 $\leftarrow$ Retrieval không phân biệt được phiên bản hiệu lực.
   - Giải pháp: Áp dụng Metadata Filtering (`is_active == True`).

4. **Next optimization nếu có thêm 1 giờ:**
   - Tích hợp **Query Decomposition** (tách multi-hop queries thành sub-queries).
   - Triển khai **Parent Context Expansion** (truy vấn theo child chunk, nhưng đưa toàn bộ parent chunk cho LLM sinh câu trả lời).
