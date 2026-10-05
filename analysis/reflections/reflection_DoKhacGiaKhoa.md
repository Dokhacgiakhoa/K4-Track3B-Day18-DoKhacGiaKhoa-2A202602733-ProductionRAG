# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Đỗ Khắc Gia Khoa  
**Khóa:** K4 - Track 3B  
**MSSV:** 2A202602733  
**Ngày hoàn thành:** 05/10/2026  

---

## Phần 1: Mapping bài giảng (Lecture Mapping)

Bảng đối chiếu chi tiết từng khái niệm lý thuyết cốt lõi trong bài giảng với mã nguồn thực tế đã cài đặt trong lab:

| Lecture Concept | Module | Hàm cụ thể | Observation & Phân tích |
|----------------|--------|-------------|--------------------------|
| **Semantic Chunking** | M1 | `chunk_semantic()` | Dùng cosine similarity giữa các vector embedding câu liên tiếp (`all-MiniLM-L6-v2`) với ngưỡng `threshold=0.85`. Khác với chunking theo độ dài cố định làm đứt gãy ý nghĩa, semantic chunking gom các câu có tính mạch lạc ngữ nghĩa cao vào cùng một khối, giảm thiểu tình trạng phân mảnh ngữ cảnh. |
| **Hierarchical Chunking (Parent-Child)** | M1 | `chunk_hierarchical()` | Tách tài liệu thành các khối lớn Parent (2048 chars) và chia nhỏ thành các Child (256 chars) mang `parent_id`. Thiết kế này giải quyết bài toán mâu thuẫn kinh điển trong RAG: Child chunk nhỏ mang vector sắc nét giúp retrieval đạt precision cao, sau đó mở rộng ngữ cảnh về Parent chunk khi đưa vào LLM để đảm bảo context window đủ thông tin. |
| **Structure-Aware Chunking** | M1 | `chunk_structure_aware()` | Parse các cấp độ Markdown headers (`#`, `##`, `###`) để phân đoạn tài liệu theo cấu trúc logic tự nhiên của tài liệu, đồng thời tự động lưu tên mục vào metadata `section`. Nhờ đó, bảo toàn nguyên vẹn các bảng biểu, danh sách, và code blocks mà không bị cắt ngang giữa chừng. |
| **Vietnamese Word Segmentation** | M2 | `segment_vietnamese()` | Sử dụng thư viện `underthesea` để tách từ tiếng Việt và chuẩn hóa thay thế dấu gạch dưới `_` thành khoảng trắng. Điều này cực kỳ quan trọng đối với thuật toán BM25: nếu giữ nguyên `nghỉ_phép` mà người dùng search `nghỉ phép`, BM25 sẽ không thể match từ vựng nếu không xử lý khoảng trắng đồng nhất. |
| **BM25 + Dense Fusion (RRF)** | M2 | `reciprocal_rank_fusion()` | Áp dụng công thức Reciprocal Rank Fusion: $RRF(d) = \sum \frac{1}{k + rank(d) + 1}$ với hằng số chuẩn hóa $k=60$. RRF giải quyết bài toán bất tương thích về thang đo điểm số giữa Dense search (Cosine similarity [-1, 1]) và BM25 (điểm không giới hạn), mang lại danh sách xếp hạng cân bằng giữa từ khóa chính xác và ngữ nghĩa tương đồng. |
| **Cross-Encoder Reranking** | M3 | `CrossEncoderReranker.rerank()` | Sử dụng model chuyên dụng `BAAI/bge-reranker-v2-m3`. Reranker tiếp nhận cặp `(query, document)` và cho phép full cross-attention giữa mọi token của truy vấn và văn bản, giúp sàng lọc chính xác từ top-20 ứng viên xuống top-3 ngữ cảnh tối ưu, cải thiện vượt trội `Context Precision`. |
| **RAGAS 4 Metrics Framework** | M4 | `evaluate_ragas()` | Đánh giá toàn diện 4 trụ cột chất lượng của RAG: **Faithfulness** (độ trung thực, không bịa đặt so với context), **Answer Relevancy** (mức độ sát với câu hỏi), **Context Precision** (tỷ lệ chunk liên quan ở thứ hạng cao), và **Context Recall** (mức độ bao phủ thông tin ground truth). |
| **Failure Analysis & Diagnostic Tree** | M4 | `failure_analysis()` | Xây dựng cây chẩn đoán tự động phân loại lỗi: Metric nào thấp nhất sẽ chỉ ra điểm nghẽn tương ứng trong kiến trúc (Faithfulness thấp -> LLM hallucination; Context Recall thấp -> Thiếu chunk, lỗi chunking/search; Context Precision thấp -> Lọt nhiều chunk rác, cần rerank). |
| **Contextual Prepend & Enrichment** | M5 | `_enrich_single_call()` & `contextual_prepend()` | Theo kỹ thuật Contextual Retrieval của Anthropic, gắn thêm thông tin tóm tắt và vị trí xuất xứ vào đầu mỗi chunk trước khi embed. Đồng thời tích hợp chế độ Combined Single-Call (1 prompt duy nhất trích xuất summary, hyqa questions, context và auto metadata), giúp tiết kiệm 75% chi phí API so với 4 cuộc gọi riêng lẻ. |

---

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

- **Lỗi kỹ thuật gặp phải (Exact error message):**
  1. `D:\Github\...\.venv\Scripts\python.exe: No module named pip`: Môi trường ảo Python 3.11 được khởi tạo bằng `uv venv` mặc định không cài sẵn `pip` truyền thống.
  2. `transformers>=5.0` không tương thích với FlagEmbedding gây lỗi `XLMRobertaTokenizer`.
  3. `qdrant-client` phiên bản mới deprecate phương thức cũ và yêu cầu dùng `query_points()`.
- **Nguyên nhân gốc rễ & Cách debug:**
  1. **Khắc phục lỗi môi trường pip**: Thay vì dùng `python -m pip install`, chuyển sang dùng trực tiếp công cụ hiện đại `uv pip install -r requirements.txt --python .venv\Scripts\python.exe`. Thời gian resolve và tải 111 packages chỉ mất chưa đầy 2 phút.
  2. **Tránh xung đột thư viện Reranker**: Tuân thủ triệt để hướng dẫn kiến trúc, sử dụng `sentence_transformers.CrossEncoder("BAAI/bge-reranker-v2-m3")` thay vì gọi qua `FlagEmbedding`, giúp model load trơn tru trên PyTorch 2.x và Transformers hiện hành.
  3. **Tương thích Qdrant Client**: Triển khai `query_points(collection_name=..., query=..., limit=...)` kèm fallback `client.search` an toàn để tương thích với mọi phiên bản server Qdrant v1.9 - v2.0.
- **Kiến thức còn thiếu & Cách khắc phục:**
  - *Hiểu sâu về độ trễ (Latency Bottleneck)*: Khi đo lường Latency Breakdown, nhận thấy Reranking bằng Cross-Encoder trên CPU chiếm tới ~97% tổng thời gian truy vấn (~11.7 giây/query).
  - *Giải pháp tối ưu*: Trong môi trường production thực tế, cần chuyển CrossEncoder sang GPU hoặc chuyển đổi sang các mô hình Onnx / Flashrank (`FlashrankReranker`) nhẹ hơn để giảm latency xuống dưới 100ms.

---

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

Dựa trên toàn bộ kinh nghiệm thực chiến từ Lab 18, tôi xây dựng kế hoạch nâng cấp kiến trúc Production RAG cho dự án cá nhân:

### Project: Trợ Lý Pháp Lý & Quy Chế Doanh Nghiệp Tự Động (Legal & Enterprise Policy Assistant)

#### 1. Hiện trạng
- **Pipeline hiện tại:** Sử dụng Basic RAG cơ bản với naive chunking theo đoạn văn, embedding đơn lẻ trên OpenAI `text-embedding-3-small`, lưu trữ trong PostgreSQL pgvector và truy vấn vector tương đồng cosine thuần túy (Dense-only).
- **Vấn đề / Bottlenecks đang gặp:**
  - *Temporal Conflict*: Nhân viên hỏi quy chế năm 2024 nhưng hệ thống trích nhầm quy chế cũ năm 2022/2023 dẫn đến câu trả lời sai luật.
  - *Low Recall trên từ khóa pháp lý chính xác*: Truy vấn các điều khoản có mã hiệu số/quyết định (ví dụ: "Nghị định 13/2023/NĐ-CP", "Điều 15 khoản 2") thường bị vector search bỏ lọt.
  - *Hallucination khi tính toán chế độ*: LLM tự tính toán nhầm số ngày phép và số tiền phạt pro-rata.

#### 2. Kế hoạch cải tiến
1. **Chunking Strategy:** Áp dụng **Hierarchical Chunking (Parent-Child)** kết hợp **Structure-Aware Chunking**. Tách theo cấu trúc Chương / Điều / Khoản của văn bản pháp quy. Child chunk (256 chars) dùng để index tìm kiếm, khi sinh câu trả lời sẽ retrieve toàn bộ Parent chunk (2048 chars) chứa nguyên vẹn điều khoản để bảo đảm ngữ cảnh pháp lý.
2. **Search Retrieval:** Bắt buộc áp dụng **Hybrid Search (BM25 tiếng Việt qua underthesea + Dense BAAI/bge-m3)** hợp nhất bằng **RRF ($k=60$)**. BM25 sẽ giải quyết triệt để việc tìm chính xác mã số văn bản, điều luật; Dense search giải quyết tính tương đồng ngữ nghĩa.
3. **Reranking:** Tích hợp **Cross-Encoder Reranker** (`BAAI/bge-reranker-v2-m3` tối ưu hóa bằng ONNX Runtime) để lọc top-20 ứng viên xuống top-3 văn bản xác đáng nhất trước khi đưa vào context prompt.
4. **Enrichment Pipeline:** Áp dụng **Combined Single-Call Enrichment** để gắn `status: active | deprecated`, `effective_date`, `scope` vào metadata của từng chunk, đồng thời bổ sung Contextual Prepend giải thích vị trí của điều khoản trong hệ thống văn bản.
5. **Evaluation:** Đưa framework **RAGAS** vào CI/CD pipeline để chạy tự động 4 metrics trên bộ test benchmark định kỳ trước mỗi đợt release dữ liệu quy chế mới.

#### 3. Timeline triển khai (4 Tuần)
- **Tuần 1: Data Processing & Ingestion**: Viết parser trích xuất cấu trúc văn bản pháp lý, áp dụng Hierarchical Chunking và gán metadata phiên bản hiệu lực.
- **Tuần 2: Hybrid Search & Vector DB**: Dựng Qdrant / Hybrid Search Engine, cấu hình bộ tách từ tiếng Việt underthesea cho BM25 và tích hợp RRF.
- **Tuần 3: Reranking & Latency Optimization**: Triển khai Cross-Encoder và benchmark tối ưu hóa tốc độ suy luận bằng TensorRT / ONNX Runtime.
- **Tuần 4: Evaluation, CI/CD & Production Launch**: Thiết lập bộ test set 50 câu hỏi nghiệp vụ, chạy RAGAS benchmark đảm bảo các metrics đạt ≥ 0.85 trước khi triển khai chính thức.
