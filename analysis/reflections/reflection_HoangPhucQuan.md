# Individual Reflection — Lab 18: Production RAG

**Tên:** Hoàng Phúc Quân - 2A202600560
**Module phụ trách:** M1 + M2 + M3 + M4 + M5 (cá nhân)

---

## 1. Đóng góp kỹ thuật

- **Modules đã implement:** M1 Chunking, M2 Hybrid Search, M3 Reranking, M4 RAGAS Eval, M5 Enrichment
- **Các hàm/class chính đã viết:**
  - M1: `chunk_semantic()`, `chunk_hierarchical()`, `chunk_structure_aware()`
  - M2: `segment_vietnamese()`, `BM25Search.index()`, `BM25Search.search()`, `DenseSearch.index()`, `DenseSearch.search()`, `reciprocal_rank_fusion()`
  - M3: `CrossEncoderReranker._load_model()`, `CrossEncoderReranker.rerank()`
  - M4: `evaluate_ragas()`, `failure_analysis()`
  - M5: `_enrich_single_call()` (combined mode)
- **Số tests pass:** 13/13 (M1) · 5/5 (M2) · 5/5 (M3) · 4/4 (M4) · 10/10 (M5)

---

## 2. Kiến thức học được

- **Khái niệm mới nhất:** Reciprocal Rank Fusion (RRF) — cách merge hai ranked list từ BM25 và Dense bằng công thức `1/(k + rank + 1)` mà không cần normalize score. Đơn giản nhưng hiệu quả vì không phụ thuộc scale của score.

- **Điều bất ngờ nhất:** Production RAG không nhất thiết tốt hơn baseline. Context Precision tăng (nhờ reranking) nhưng Faithfulness và Context Recall đều giảm. Pipeline phức tạp hơn tạo ra nhiều điểm có thể fail hơn — đặc biệt enrichment thất bại ~12% chunks do LLM trả về markdown code block thay vì raw JSON.

- **Kết nối với bài giảng:**
  - Slide về "Contextual Retrieval" (Anthropic paper): `contextual_prepend()` trong M5 — thêm 1 câu prefix mô tả chunk nằm ở đâu trong tài liệu, benchmark giảm 49% retrieval failure.
  - Slide về "Hybrid Search": RRF trong M2 là implementation trực tiếp của công thức trong slide.
  - Slide về "RAGAS Diagnostic Tree": mapping `worst_metric → diagnosis → fix` trong `failure_analysis()` M4.

---

## 3. Lecture → Code Mapping

| Lecture Concept | Module | Hàm cụ thể | Observation |
|----------------|--------|-------------|-------------|
| Semantic chunking | M1 | `chunk_semantic()` | Threshold 0.85 tạo chunks ít hơn nhưng coherent hơn. Dùng `all-MiniLM-L6-v2` nhanh hơn `bge-m3` 3×, đủ cho sentence-level similarity. |
| Hierarchical chunking | M1 | `chunk_hierarchical()` | Parent 2048 + Child 256. Child 256 chars quá nhỏ → context_recall giảm vì bảng phân quyền bị cắt. Nên dùng 512. |
| BM25 + Dense fusion | M2 | `reciprocal_rank_fusion()` | RRF với k=60 giải quyết scale mismatch giữa BM25 score (float tùy ý) và cosine score (0-1). |
| Vietnamese tokenization | M2 | `segment_vietnamese()` | underthesea nối từ ghép bằng `_` → phải `replace("_", " ")` để BM25 tokenize đúng. Bỏ bước này là silent bug. |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | `BAAI/bge-reranker-v2-m3` latency ~40ms/query (5 docs). Tăng context_precision từ 0.933 → 0.975 nhưng top-3 quá ít cho multi-hop questions. |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | context_precision cao nhất (0.975) — reranking work. faithfulness thấp nhất (0.610) — LLM hallucinate. |
| Contextual embeddings | M5 | `_enrich_single_call()` | Combined 1-call mode tiết kiệm 4× API calls. Bug: LLM trả về ```json code block → json.loads() fail. Fix: strip markdown trước khi parse. |

---

## 4. Khó khăn & Cách giải quyết

**Khó khăn 1: M5 Enrichment JSON parse error**
- **Error:** `Expecting value: line 1 column 1 (char 0)` — xuất hiện ~12 lần trong 100 chunks
- **Debug:** Print raw response → LLM trả về ` ```json\n{...}\n``` ` thay vì raw JSON
- **Fix:** Thêm `re.sub(r'```json\s*|\s*```', '', resp_text).strip()` trước `json.loads()`
- **Thời gian debug:** ~15 phút (phát hiện sau khi xem output trực tiếp từ OpenAI)

**Khó khăn 2: Qdrant `recreate_collection` deprecated**
- **Error:** `AttributeError: 'QdrantClient' object has no attribute 'recreate_collection'`
- **Fix:** Dùng `delete_collection()` + `create_collection()` hoặc `client.recreate_collection()` với qdrant-client version cũ hơn
- **Thời gian debug:** ~10 phút

**Khó khăn 3: Context Recall giảm khi chuyển sang hierarchical chunking**
- **Observation:** Naive (paragraph, ~57 chunks) có context_recall=0.925 cao hơn Production (hierarchical, 100 child chunks) = 0.792
- **Root cause:** Child chunk 256 chars cắt giữa bảng/danh sách — thông tin quan trọng bị split
- **Kiến thức thiếu:** Parent retrieval pattern: retrieve child để match, nhưng trả về text của parent để có context đầy đủ. Pipeline hiện tại chỉ dùng child text.

---

## 5. Nếu làm lại

**Sẽ làm khác:**
- Fix enrichment JSON bug ngay từ đầu (thêm JSON extraction với regex)
- Tăng HIERARCHICAL_CHILD_SIZE từ 256 → 512 chars
- Implement parent retrieval: sau khi rerank child chunks, swap text sang parent text trước khi send vào LLM
- Thêm version metadata vào corpus có nhiều version (mat_khau_v1/v2, nghi_phep v2023/v2024)

**Module muốn thử tiếp:**
- Implement FlashrankReranker (M3) để so sánh latency: bge-reranker (~40ms) vs flashrank (<5ms) với accuracy trade-off
- Thử HyDE (Hypothetical Document Embeddings) trong M5 thay vì HyQA

---

## 6. Action Plan cho Project

### Hiện tại
- RAG pipeline hiện tại: chưa có production RAG, đang ở mức prototype
- Known issues: chunking naive (paragraph), không có reranking, không evaluate

### Plan áp dụng
1. **Chunking:** Dùng `chunk_structure_aware()` cho tài liệu Markdown có headers rõ ràng. Dùng `chunk_hierarchical()` với child_size=512 cho PDF text.
2. **Search:** Hybrid BM25 + Dense với RRF. BM25 quan trọng cho tiếng Việt vì tên riêng và số liệu.
3. **Reranking:** `bge-reranker-v2-m3` cho accuracy. Chuyển sang `flashrank` nếu latency là bottleneck.
4. **Evaluation:** RAGAS với 4 metrics. Focus vào faithfulness (ngăn hallucination) và context_recall (đảm bảo không miss thông tin).
5. **Enrichment:** Combined mode (1 call/chunk) với JSON extraction bug fixed. Ưu tiên `contextual_prepend` vì có benchmark 49% improvement từ Anthropic.

### Timeline
- Tuần 1: Fix enrichment + parent retrieval, re-run pipeline → đo lại scores
- Tuần 2: Thêm version metadata + Qdrant filter cho docs đa version
- Tuần 3: Evaluation on full test set (30 questions) + failure analysis

---

## 7. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4 |
| Code quality | 4 |
| Teamwork | N/A (cá nhân) |
| Problem solving | 4 |


