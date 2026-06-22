# Failure Analysis — Lab 18: Production RAG

**Thành viên:** Hoàng Phúc Quân - 2A202600560

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.7383 | 0.6100 | -0.1283 ↓ |
| Answer Relevancy | 0.5763 | 0.4531 | -0.1232 ↓ |
| Context Precision | 0.9333 | 0.9750 | +0.0417 ↑ |
| Context Recall | 0.9250 | 0.7917 | -0.1333 ↓ |

> **Nhận xét tổng quan:** Context Precision cải thiện (+4.2%) nhờ hierarchical chunking + cross-encoder reranking.
> Tuy nhiên, 3 metrics còn lại đều giảm — đặc biệt Faithfulness và Context Recall giảm đáng kể.
> Nguyên nhân chính: (1) enrichment thất bại ~12% chunks do JSON parse error, (2) hierarchical child chunks
> 256 chars quá nhỏ làm mất context, (3) LLM hallucinate trên câu hỏi phủ định và xung đột version.

---

## Bottom-5 Failures

### #1 — avg_score: 0.4167

- **Question:** Mentor và buddy của nhân viên mới có thể là cùng một người không? Quản lý trực tiếp có thể làm mentor không?
- **Expected:** KHÔNG cho cả hai. Mentor và buddy phải là hai người khác nhau. Quản lý trực tiếp không được làm mentor hoặc buddy.
- **Worst metric:** faithfulness = 0.0
- **Error Tree:** Output sai → Context có đủ không? (có, mentor_buddy.md được index) → LLM dùng context không? → LLM thêm thông tin ngoài context (hallucinate điều kiện ngoại lệ không có trong tài liệu)
- **Root cause:** Câu hỏi có 2 phần (AND logic). LLM cố gắng trả lời trọn vẹn nên suy diễn thêm thông tin không có trong context. Enrichment chunk này thất bại nên thiếu context prefix hỗ trợ.
- **Suggested fix:** Tighten system prompt: "Chỉ trả lời những gì CÓ trong context. Nếu context không đủ → nói rõ 'Tài liệu không đề cập'." Giảm temperature xuống 0.

---

### #2 — avg_score: 0.4583

- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Theo chính sách hiện hành (v2.0), mật khẩu phải được thay đổi mỗi 120 ngày. Chính sách cũ yêu cầu 90 ngày nhưng đã bị thay thế.
- **Worst metric:** faithfulness = 0.0
- **Error Tree:** Output sai → Context đúng? → Corpus có cả mat_khau_v1.md (90 ngày) và mat_khau_v2.md (120 ngày) → Reranker trả về cả hai → LLM chọn nhầm version cũ hoặc trung bình hóa
- **Root cause:** Version conflict — hai file chính sách mật khẩu tồn tại song song. BM25 + Dense đều retrieve cả v1 và v2. Cross-encoder không có tín hiệu phân biệt "hiện hành" vs "cũ". LLM bị confuse và nêu số 90 ngày (từ v1) dù context có v2.
- **Suggested fix:** Thêm metadata `is_current: true/false` vào enrichment. Dùng metadata filter trong Qdrant khi search. Hoặc thêm instruction vào prompt: "Ưu tiên chính sách có version mới nhất."

---

### #3 — avg_score: 0.4583

- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Đơn hàng trên 50.000.000 VNĐ cần Tổng Giám đốc (CEO) phê duyệt.
- **Worst metric:** context_recall = 0.0
- **Error Tree:** Output sai → Context đúng? → context_recall=0 nghĩa là chunk liên quan KHÔNG được retrieve → Query "55 triệu" không khớp với text "trên 50.000.000 VNĐ" → BM25 miss, Dense search cũng miss vì số tiền khác nhau
- **Root cause:** Retrieval failure hoàn toàn. Thông tin "CEO phê duyệt đơn >50M" nằm trong một child chunk 256 chars, bị tách khỏi bảng phân quyền mua sắm gốc. BM25 không match "55 triệu" với "50.000.000". Dense search fail vì semantic gap giữa giá trị cụ thể và ngưỡng chính sách.
- **Suggested fix:** Dùng parent-level retrieval: khi child chunk match, trả về cả parent chunk (2048 chars) để có đầy đủ bảng phân quyền. Hoặc tăng HIERARCHICAL_CHILD_SIZE lên 512 chars để bảng không bị cắt giữa chừng.

---

### #4 — avg_score: 0.5000

- **Question:** Nhân viên thử việc có được nghỉ phép năm không?
- **Expected:** KHÔNG. Nhân viên thử việc KHÔNG được nghỉ phép năm. Nếu cần nghỉ, phải xin nghỉ không lương và được trưởng phòng phê duyệt.
- **Worst metric:** faithfulness = 0.0
- **Error Tree:** Output sai → Context đúng? (có thể đúng) → LLM thêm thông tin ngoài context → Negation question: LLM xu hướng "softened" câu trả lời phủ định
- **Root cause:** LLM hallucinate trên câu hỏi phủ định. Thay vì trả lời dứt khoát "KHÔNG", LLM thường thêm điều kiện như "tùy trường hợp" hoặc "có thể xin phép đặc biệt" — những thông tin không có trong tài liệu. Đây là bias phổ biến của LLM với negation.
- **Suggested fix:** Thêm instruction vào prompt: "Với câu hỏi có/không, trả lời CÓ hoặc KHÔNG trước, sau đó giải thích. Không thêm thông tin ngoài context."

---

### #5 — avg_score: 0.5000

- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** KHÔNG. Nhân viên thử việc chưa được hưởng gói bảo hiểm sức khỏe PVI. Chỉ được tham gia bảo hiểm xã hội bắt buộc.
- **Worst metric:** faithfulness = 0.0
- **Error Tree:** Output sai → Context đúng? → Query "thử việc + bảo hiểm PVI" match chunk bao_hiem_suc_khoe.md → LLM có context → LLM thêm thông tin về BHXH bắt buộc mà không có trong context được retrieve hoặc ngược lại
- **Root cause:** Tương tự #4 — negation question với LLM hallucination. Ngoài ra, enrichment chunk bao_hiem_suc_khoe.md có thể thất bại (JSON error), khiến context prefix bị thiếu, LLM không biết chunk này từ tài liệu nào để trả lời chính xác.
- **Suggested fix:** Thêm source citation vào context: "Từ tài liệu [bao_hiem_suc_khoe.md]: ...". Giảm temperature = 0 cho câu hỏi policy.

---

## Case Study (Phân tích chuyên sâu)

**Question chọn phân tích:** "Bao lâu phải đổi mật khẩu một lần?" (#2)

**Error Tree walkthrough:**
1. **Output đúng?** → KHÔNG (LLM trả lời 90 ngày thay vì 120 ngày)
2. **Context đúng?** → Context_precision cao (0.975) nhưng context có CẢ HAI version — v1 (90 ngày) và v2 (120 ngày) cùng được retrieve
3. **Query rewrite OK?** → BM25 segment "đổi mật khẩu" match tốt cả v1 và v2, Dense search cũng tương tự → không phân biệt được
4. **Fix ở bước:** Metadata enrichment — cần gắn tag `{version: "v2", is_current: true}` vào mat_khau_v2.md và `{is_current: false}` vào v1. Dùng Qdrant filter `must: [{key: "is_current", match: {value: true}}]` hoặc boost score cho version mới hơn trong RRF.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Fix enrichment JSON parse error: LLM trả về markdown code block ` ```json ... ``` ` thay vì raw JSON. Cần strip bằng `re.sub(r'```json\s*|\s*```', '', response)` trước khi `json.loads()`.
- Tăng HIERARCHICAL_CHILD_SIZE từ 256 → 512 chars để giảm context_recall drop.
- Thêm version-aware metadata filter vào Qdrant search cho các doc có nhiều version.
- Thêm câu lệnh phủ định vào system prompt để giảm hallucination trên yes/no questions.
