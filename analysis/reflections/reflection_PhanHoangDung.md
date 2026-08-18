# Individual Reflection — Lab 18: Production RAG Pipeline

**Họ và tên:** Phan Hoàng Dũng  
**MSSV:** 2A202601348  
**Module phụ trách:** Toàn bộ Pipeline (M1 Chunking, M2 Hybrid Search, M3 Reranking, M4 Evaluation, M5 Enrichment)

---

## Phần 1: Mapping bài giảng (Lecture → Code)

| Lecture Concept | Module | Hàm cụ thể | Observation & Phân tích chuyên sâu |
|---|:---:|---|---|
| **Semantic Chunking** | M1 | `chunk_semantic()` | Dùng cosine similarity giữa các vector câu liên tiếp với ngưỡng `threshold = 0.85`. Khác với naive fixed-size, kỹ thuật này giữ trọn vẹn mạch ý nghĩa và ngữ cảnh logic của từng chủ đề. |
| **Hierarchical Chunking** | M1 | `chunk_hierarchical()` | Tạo cấu trúc Parent (2048 chars) $\rightarrow$ Child (256 chars). Retrieve bằng child chunk nhằm tối ưu hóa độ chính xác từ khóa/ngữ nghĩa, sau đó trả về parent context để LLM có đầy đủ thông tin bối cảnh. |
| **Structure-Aware Chunking** | M1 | `chunk_structure_aware()` | Phân tích cấu trúc tiêu đề Markdown (`#`, `##`, `###`), bảo toàn bảng biểu và danh sách nguyên vẹn trong cùng 1 section, tránh việc cắt đôi bảng dữ liệu phân quyền hay thang bảng lương. |
| **Vietnamese Word Segmentation** | M2 | `segment_vietnamese()` | Sử dụng `underthesea.word_tokenize` và chuẩn hóa khoảng trắng (`replace("_", " ")`) để thuật toán BM25 nhận diện đúng các từ ghép tiếng Việt (ví dụ: "nghỉ phép", "bảo hiểm") thay vì bị lệch token với truy vấn. |
| **Hybrid Search & RRF Fusion** | M2 | `reciprocal_rank_fusion()` | Kết hợp BM25 (keyword match) và Dense BAAI/bge-m3 (semantic match) thông qua hàm tính điểm $RRF = \sum \frac{1}{k + rank + 1}$ với $k=60$. Giúp giải quyết triệt để bài toán từ đồng nghĩa, từ viết tắt và out-of-vocabulary. |
| **Cross-Encoder Reranking** | M3 | `CrossEncoderReranker.rerank()` | Áp dụng mô hình `BAAI/bge-reranker-v2-m3` tính toán ma trận cross-attention đầy đủ giữa `(query, document)`. Rerank từ top-20 xuống top-3 giúp loại bỏ các false-positive chunks, đạt Context Precision ấn tượng **0.9208**. |
| **RAGAS 4 Metrics Evaluation** | M4 | `evaluate_ragas()` | Đánh giá tự động toàn diện qua 4 khía cạnh cốt lõi: `Faithfulness` (0.69), `Answer Relevancy` (0.76), `Context Precision` (0.92), `Context Recall` (0.83). |
| **Diagnostic Tree Failure Analysis** | M4 | `failure_analysis()` | Tự động phân loại lỗi của hệ thống RAG theo cây chẩn đoán (Diagnostic Tree), xác định điểm nghẽn nằm ở khâu Retriever (thiếu chunk), Reranker (nhiễu context), hay Generator (hallucination). |
| **Contextual Prepend & HyQA** | M5 | `_enrich_single_call()` / `enrich_chunks()` | Làm giàu tài liệu trước khi embed bằng cách sinh 3 câu hỏi giả định (HyQA), tạo tóm tắt ngắn, trích xuất metadata và thêm câu dẫn bối cảnh (Anthropic style) giúp giảm thiểu retrieval failure. |

---

## Phần 2: Khó khăn & Cách giải quyết

### 1. Lỗi gặp phải (Exact Error & Gotchas)
- **Lỗi 1 (BM25 Token Mismatch):** `underthesea.word_tokenize(..., format="text")` sinh ra các từ nối bởi dấu gạch dưới `_` (ví dụ: `"nghỉ_phép"`). Khi BM25 tokenizer tách chuỗi theo khoảng trắng `" "`, chuỗi `"nghỉ_phép"` trở thành 1 token duy nhất. Tuy nhiên người dùng nhập query `"nghỉ phép"` (2 tokens tách rời), dẫn đến BM25 không tính điểm trùng khớp.
  - *Cách fix:* Thực hiện `segmented.replace("_", " ")` trước khi đưa vào BM25 indexing và search.
- **Lỗi 2 (Cross-Encoder / Transformers compatibility):** Khi sử dụng `FlagEmbedding.FlagReranker`, thư viện xảy ra lỗi xung đột tokenization (`XLMRobertaTokenizer` crash) trên môi trường `transformers >= 5.0`.
  - *Cách fix:* Chuyển sang sử dụng `sentence_transformers.CrossEncoder("BAAI/bge-reranker-v2-m3")` chạy ổn định, tối ưu bộ nhớ.
- **Lỗi 3 (Temporal Version Conflicts & Multi-hop Recall):** Khi tập dữ liệu chứa chính sách cũ (v2023) và chính sách mới (v2024), việc rút gọn context chỉ lấy top-3 khiến LLM đôi khi nhận trúng chunk phiên bản cũ hoặc thiếu 1 trong 2 vế của câu hỏi ghép (compound query).
  - *Cách fix:* Xác định nguyên nhân qua Diagnostic Tree, đề xuất giải pháp thêm Metadata Filter (`version`) và Query Decomposition.

### 2. Cách debug & Quá trình giải quyết
- Đọc kỹ log lỗi chi tiết từ terminal và chạy từng unit test tương ứng trong thư mục `tests/`.
- In thử cấu trúc token sau khi tiền xử lý tiếng Việt để so sánh giữa corpus vocabulary và query tokens.
- Phân tích chi tiết file kết quả `ragas_report.json` qua 10 câu hỏi có điểm số thấp nhất để chỉ ra chính xác khâu nào trong pipeline cần tinh chỉnh.

### 3. Kiến thức tích lũy thêm
- Hiểu sâu bản chất sự khác nhau giữa Bi-Encoder (Dense Search - embedding độc lập, nhanh nhưng tương tác nông) và Cross-Encoder (Reranker - full attention giữa câu hỏi và văn bản, chính xác cao nhưng tốn chi phí tính toán).
- Nắm vững quy trình đánh giá chất lượng RAG định lượng theo chuẩn RAGAS thay vì đánh giá cảm tính.

---

## Phần 3: Action Plan áp dụng vào Project

### Project: Hệ thống Trợ lý Pháp lý & Tra cứu Quy chế Doanh nghiệp Thông minh

### Hiện tại
- **RAG pipeline hiện tại:** Sử dụng Naive RAG cơ bản với RecursiveCharacterTextSplitter (chunk size 500, overlap 50), OpenAI `text-embedding-3-small`, tìm kiếm Cosine Similarity đơn thuần trên ChromaDB, sinh câu trả lời trực tiếp bằng GPT-4o-mini.
- **Known issues:**
  1. Thường xuyên trả lời nhầm các văn bản pháp quy / quy chế đã hết hiệu lực do không phân biệt được phiên bản (Temporal ambiguity).
  2. Không trả lời được các câu hỏi so sánh hoặc câu hỏi tổng hợp nhiều điều luật (Multi-hop failure).
  3. Bị cắt vụn các điều khoản có chứa bảng phụ lục và danh sách điều kiện.

### Plan áp dụng kiến trúc Production RAG

1. **Chunking Strategy:**
   - Áp dụng **Structure-Aware Chunking** theo cấu trúc Điều/Khoản/Điểm của văn bản quy phạm pháp luật kết hợp **Hierarchical Parent-Child (2048 / 256)** để bảo toàn toàn bộ ngữ cảnh điều khoản.
2. **Search Strategy:**
   - Triển khai **Hybrid Search**: BM25 (đã segment tiếng Việt qua `underthesea`) + Dense Search (`BAAI/bge-m3` đa ngôn ngữ) kết hợp thuật toán **RRF (k=60)** để bắt chính xác số hiệu văn bản (ví dụ: "Nghị định 123/2024/NĐ-CP").
3. **Reranking:**
   - Tích hợp **Cross-Encoder Reranker (`bge-reranker-v2-m3`)** lấy top-25 candidate xuống top-4 trước khi đưa vào prompt generator.
4. **Enrichment:**
   - Sử dụng **Single-Call Enrichment** để sinh HyQA, tóm tắt điều khoản và tự động trích xuất metadata: `so_hieu`, `ngay_ban_hanh`, `ngay_hieu_luc`, `trang_thai_hieu_luc` (Còn hiệu lực / Hết hiệu lực).
5. **Query Processing & Filtering:**
   - Thêm bước **Query Decomposition** cho các câu hỏi phức hợp và áp dụng **Metadata Pre-filtering** theo trạng thái hiệu lực của văn bản.
6. **Evaluation:**
   - Tích hợp pipeline đánh giá tự động định kỳ với **RAGAS** (4 metrics) trên bộ test set 50 câu hỏi chuẩn hóa của doanh nghiệp.

### Timeline triển khai

| Tuần | Mục tiêu & Hạng mục công việc | Deliverables |
|:---:|---|---|
| **Tuần 1** | Tái cấu trúc pipeline dữ liệu: Áp dụng Structure-aware & Hierarchical chunking, tiền xử lý metadata văn bản pháp quy. | Bộ chunks chuẩn hóa kèm metadata `version`, `effective_date`. |
| **Tuần 2** | Nâng cấp Search Engine: Dựng Hybrid Search (BM25 + Qdrant BGE-M3 + RRF), tích hợp Cross-Encoder Reranker. | Hybrid Retriever module đạt Context Precision $\ge 0.90$. |
| **Tuần 3** | Triển khai Enrichment Pipeline (HyQA, Context Prepend) và Module Query Decomposition (Sub-queries). | Hệ thống xử lý mượt mà câu hỏi multi-hop và bảng biểu. |
| **Tuần 4** | Chạy benchmark RAGAS trên 50 câu test cases, phân tích Diagnostic Tree, tối ưu prompt template và triển khai production. | Báo cáo đánh giá RAGAS (tất cả 4 metrics $\ge 0.85$). |

---

## Phần 4: Tự đánh giá

| Tiêu chí | Tự chấm (1-5) | Ghi chú minh chứng |
|---|:---:|---|
| **Hiểu bài giảng & Lý thuyết** | 5/5 | Nắm vững toàn bộ 5 modules, giải thích rõ nguyên lý RRF, Cross-Encoder và RAGAS. |
| **Chất lượng Code (Code Quality)** | 5/5 | Hoàn thiện 100% TODOs, code clean, có type hinting, logging, exception handling và fallback đầy đủ. |
| **Tư duy giải quyết vấn đề (Problem Solving)** | 5/5 | Phân tích sâu sắc nguyên nhân gốc rễ (Root Cause) của top-5 failures và đưa ra giải pháp kỹ thuật cụ thể. |
| **Kế hoạch ứng dụng thực tiễn (Action Plan)** | 5/5 | Action plan chi tiết, có timeline rõ ràng và gắn liền với bài toán thực tế của project. |
