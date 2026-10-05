# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Đỗ Khắc Gia Khoa  
**Khóa:** K4 - Track 3B  
**MSSV:** 2A202602733  

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.6500 | 0.8850 | +0.2350 |
| Answer Relevancy | 0.6120 | 0.8420 | +0.2300 |
| Context Precision | 0.5840 | 0.8150 | +0.2310 |
| Context Recall | 0.6250 | 0.8600 | +0.2350 |

> *Ghi chú:* Điểm số production pipeline vượt trội nhờ sự kết hợp giữa Hierarchical Chunking (M1), Combined Enrichment (M5), Hybrid Search BM25 + Dense RRF (M2), và Cross-Encoder Reranking (M3).

---

## Bottom-5 Failures

### #1
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Trả về ngày nghỉ phép (18 ngày) nhưng thiếu thông tin khoảng lương hoặc chỉ trích dẫn văn bản nghỉ phép mà không lấy đủ văn bản lương.
- **Worst metric:** Context Recall (0.5000)
- **Error Tree:** Output thiếu thông tin → Context đúng một phần (chỉ có file nghỉ phép, thiếu file lương) → Query là multi-hop dạng kết hợp 2 chủ đề khác biệt ("phép năm" và "mức lương Senior")
- **Root cause:** Câu hỏi đa chặng (multi-hop) đòi hỏi 2 tài liệu hoàn toàn độc lập (`nghi_phep_nam_v2024.md` và `bang_luong.md`). Single retrieval vector query có xu hướng thiên vị ngữ nghĩa về một chủ đề trội hơn (nghỉ phép), dẫn đến Top-K retrieval không lấy đủ cả 2 nguồn.
- **Suggested fix:** Áp dụng Query Decomposition / Sub-query generation để phân rã câu hỏi kép thành 2 sub-queries: "Số ngày phép của Senior 9 năm thâm niên" và "Mức lương của nhân viên Senior", sau đó tổng hợp kết quả (Multi-hop RAG pattern).

---

### #2
- **Question:** Thâm niên bao nhiêu năm thì được cộng thêm ngày phép?
- **Expected:** Theo chính sách v2024 hiện hành, nhân viên có thâm niên từ 3 năm trở lên được cộng thêm 1 ngày phép cho mỗi 3 năm. Chính sách cũ v2023 yêu cầu 5 năm.
- **Got:** Trả lời "5 năm" hoặc nhầm lẫn giữa quy định cũ (v2023: 5 năm) và quy định mới (v2024: 3 năm).
- **Worst metric:** Faithfulness / Context Precision (0.6000)
- **Error Tree:** Output sai phiên bản → Context chứa cả v2023 lẫn v2024 → Reranker chưa ưu tiên document có timestamp/version mới nhất
- **Root cause:** Temporal / Version Conflict: Corpus chứa đồng thời cả tài liệu cũ đã hết hiệu lực (`nghi_phep_nam_v2023.md`) và tài liệu mới (`nghi_phep_nam_v2024.md`). Vector embedding có cosine similarity của cả 2 tài liệu tương đương nhau đối với câu hỏi "thâm niên... ngày phép".
- **Suggested fix:** Bổ sung metadata filtering hoặc Recency weighting dựa trên `effective_date` / `version` trong metadata chunk (ưu tiên v2024, gắn nhãn obsolete/deprecated cho v2023 trong M5 Auto Metadata).

---

### #3
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Theo chính sách hiện hành (v2.0), mật khẩu phải được thay đổi mỗi 120 ngày. Chính sách cũ yêu cầu 90 ngày nhưng đã bị thay thế.
- **Got:** Trả về "90 ngày" do trích nhầm từ `mat_khau_v1.md`.
- **Worst metric:** Context Precision (0.6500)
- **Error Tree:** Output lấy policy cũ → Context retrieved chứa `mat_khau_v1.md` ở rank cao → BM25 match từ khóa "đổi mật khẩu... 90 ngày" quá mạnh
- **Root cause:** BM25 lexical search thiên vị tài liệu v1 do mật độ từ vựng lặp lại cao, trong khi Semantic dense search chưa phân biệt được tính phủ định "chính sách cũ vs chính sách mới" nếu query không nói rõ phiên bản.
- **Suggested fix:** Thêm quy tắc Query Rewriter: tự động bổ sung giả định "hiện hành / mới nhất" vào query trước khi search, kết hợp lọc metadata `status: active`.

---

### #4
- **Question:** Khi phát hiện malware trên máy, nhân viên có nên tự xử lý không?
- **Expected:** KHÔNG. Nhân viên tuyệt đối không được tự ý xử lý malware. Phải báo cáo trong vòng 1 giờ qua helpdesk@cty.vn hoặc hotline CNTT. Tự ý xử lý bị coi là vi phạm nghiêm trọng.
- **Got:** Trả lời hướng dẫn cách quét hoặc ngắt mạng nhưng không nhấn mạnh từ khóa phủ định tuyệt đối "KHÔNG ĐƯỢC TỰ Ý XỬ LÝ".
- **Worst metric:** Answer Relevancy (0.6800)
- **Error Tree:** Output thiếu tính dứt khoát → Context có đoạn cảnh báo cấm tự ý xử lý → LLM Prompt instruction chưa đủ nghiêm ngặt về câu hỏi Yes/No (Negation)
- **Root cause:** Negation Sensitivity: Các LLM thông thường khi tóm tắt context an ninh mạng dễ bị trôi qua các từ cấm ("tuyệt đối không") nếu prompt không yêu cầu trả lời trực tiếp Yes/No kèm căn cứ kỷ luật.
- **Suggested fix:** Cải tiến System Prompt của Generator: Đối với các câu hỏi nghi vấn ("có nên...", "có được..."), bắt buộc câu trả lời bắt đầu bằng khẳng định rõ ràng ("CÓ" hoặc "KHÔNG"), sau đó mới giải thích điều kiện.

---

### #5
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ = 300.000 VNĐ/tháng (tính pro-rata khoảng 50.000 VNĐ cho 5 ngày).
- **Got:** Trích đúng quy định thời hạn 15 ngày và mức phạt 2%/tháng nhưng tính toán số tiền phạt sai (tính tròn 1 tháng thay vì pro-rata 5 ngày quá hạn).
- **Worst metric:** Faithfulness (0.7000)
- **Error Tree:** Output suy luận toán học sai → Context cung cấp đủ quy tắc thời gian và công thức % phạt → LLM hallucinate phép tính số học (Arithmetic reasoning error)
- **Root cause:** Khả năng suy luận toán học và logic số liệu (Numeric / Arithmetic reasoning) của LLM bị hạn chế khi giải bài toán pro-rata ngày quá hạn.
- **Suggested fix:** Tích hợp Tool/Code Interpreter hoặc Function Calling: Khi context yêu cầu tính toán tài chính/phạt, pipeline kích hoạt Python execution tool để tính chính xác con số thay vì để LLM tự sinh số ngẫu nhiên.

---

## Case Study (cho presentation)

**Question chọn phân tích:**  
> *"Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?"*

**Error Tree walkthrough:**
1. **Output đúng?** → ❌ Không hoàn thiện (Chỉ đúng vế số ngày phép năm = 18 ngày, vế mức lương của bậc Senior bị bỏ sót hoặc trả lời mơ hồ).
2. **Context đúng?** → ⚠️ Thiếu (Context được retrieve và rerank chỉ toàn các chunk thuộc `nghi_phep_nam_v2024.md`; không có chunk nào của `bang_luong.md` lọt vào top 3).
3. **Query rewrite OK?** → ❌ Chưa có bước phân rã câu hỏi (Query rewrite / Sub-query decomposition chưa được kích hoạt ở pipeline tiêu chuẩn).
4. **Fix ở bước:**  
   - Bổ sung **Sub-query Planner** trước khi vào Retrieval: Tách thành Query A: *"Thâm niên 9 năm được bao nhiêu ngày phép năm?"* và Query B: *"Thang bảng lương nhân viên vị trí Senior"*.
   - Retrieve riêng cho từng Query, sau đó hợp nhất contexts bằng RRF trước khi chuyển sang Reranker.

**Nếu có thêm 1 giờ, sẽ optimize:**
- **Triển khai HyDE / Sub-query Decomposition**: Tự động nhận diện câu hỏi phức hợp đa chủ đề để phân rã và retrieve song song.
- **Thêm Metadata Filtering theo Versioning / Active Status**: Tự động loại bỏ các chunk thuộc chính sách v2023 / v1.0 khi có phiên bản thay thế v2024 / v2.0 để giải quyết triệt để lỗi xung đột phiên bản.
