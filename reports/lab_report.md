# Báo cáo thực hành & thuyết minh kỹ thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Quang Huy  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  
**Phạm vi chạy:** 1.500 bài, 1.500 chunks cho Flat RAG, 400 chunks cho KG extraction, 5 Golden queries.

---

## PHẦN 1 — THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution

- **Ví dụ thực tế:** chunk `bc242e9e7ee79ae11a3d::c0000` chứa MicroTech và đại từ sở hữu `their`. Model trả văn bản đã đổi thành `MicroTech's Veterans Technology Services 2`, nhưng đồng thời vẫn ghi `their` trong `unresolved_mentions`.
- **Hiện tượng:** output tự mâu thuẫn giữa phép thay thế và cờ “chưa phân giải”. Đây là trường hợp antecedent có vẻ hợp lý nhưng contract của model chưa được tuân thủ tuyệt đối.
- **Hậu quả đối với graph:** nếu phép gán MicroTech là sai, NER/RE có thể tạo false edge giữa MicroTech và hợp đồng VETS 2/CDC. Pipeline vì vậy giữ prompt conservative, ghi audit và không cho batch lỗi đi tiếp. Kết quả cuối: `Coreference batch failures: 0` trên 400 chunks.

### 2. Entity Resolution Threshold & Lexical Guard

- **Ngưỡng vector:** cosine similarity `0.90`; lexical merge threshold `0.86` sau khi chuẩn hóa và bỏ hậu tố doanh nghiệp.
- **Cặp bị guard chặn:** `Liverpool` và `Liverpool FC`, cosine `0.908026`, quyết định `REJECT_GUARD`.
- **Lý do:** embedding coi hai tên rất gần nhau, nhưng lexical guard nhận ra phần `FC` làm thay đổi định danh; gộp thành một entity có thể nhập nhằng thành phố/tổ chức với câu lạc bộ bóng đá. Ngược lại, `Affirm Holdings` và `Affirm Holdings Inc` đạt `0.963123` và được `MERGE_VECTOR` vì khác biệt chỉ là hậu tố doanh nghiệp.
- **Giới hạn quan sát:** ở sample hiện tại chỉ có hai cặp vượt ngưỡng candidate; audit vẫn hiển thị cả merge lẫn reject, nhưng ở production cần log thêm candidate dưới ngưỡng để có mẫu audit lớn hơn.

### 3. Đồ thị & Super-node Mitigation

Đồ thị cuối có **289 nodes**, **163 edges** và **0 cạnh thiếu provenance**.

| Hạng | Thực thể | Loại | Degree |
|---:|---|---|---:|
| 1 | Adidas | Company | 9 |
| 2 | Microsoft | Company | 4 |
| 3 | Activision Blizzard | Company | 4 |

Không node nào vượt ngưỡng super-node `degree > 100` trong sample, nên nhánh cap không bị kích hoạt thực tế. Invariant cấu hình là tối đa 50 cạnh/node và 250 cạnh cho toàn context.

- **Ưu điểm:** giới hạn context explosion, ưu tiên bằng chứng mới, giảm latency/token và tránh một công ty phổ biến lấn át toàn bộ retrieval.
- **Rủi ro:** câu hỏi lịch sử có thể mất cạnh cũ; sắp xếp theo ngày cũng thiên lệch khi ngày thiếu hoặc không chuẩn. Cách khắc phục là cap theo intent/temporal filter và dành quota cho cả cạnh mới lẫn cạnh có relevance cao.

### 4. So sánh thực nghiệm Flat RAG và GraphRAG

| Tiêu chí | Flat RAG | GraphRAG | Δ Graph − Flat | Nhận xét |
|---|---:|---:|---:|---|
| Comprehensiveness (1–5) | 4.000 | 4.600 | +0.600 | GraphRAG đầy đủ hơn, nổi bật ở multi-hop |
| Faithfulness (1–5) | 4.200 | 4.800 | +0.600 | Graph provenance giúp giảm câu trả lời sai context |
| Multi-hop Reasoning (1–5) | 4.000 | 4.600 | +0.600 | GraphRAG nối evidence tốt hơn |
| Latency trung bình (s) | 1.998 | 2.027 | +0.029 | Gần tương đương trong sample nhỏ |
| Token trung bình | 675.6 | 624.0 | -51.6 | GraphRAG dùng ít hơn 51,6 token/query trong lần chạy này |

#### Ca lỗi 1 — Flat RAG thất bại, GraphRAG thành công

- **Question:** `G5000-49`, ba domain công nghệ của Samsung.
- **Kết quả:** Flat đạt `2/2/2`; GraphRAG đạt `5/5/5` cho comprehensiveness/faithfulness/multi-hop.
- **Nguyên nhân:** các bằng chứng nằm ở nhiều records về display/biometric sensing, smart-home sensing và semiconductor. Top-k vector của Flat đưa vào một domain ICT không đúng trọng tâm, trong khi hybrid context tập hợp đúng các quan hệ/entity liên quan.

#### Ca lỗi 2 — Cả hai còn khó khăn

- **Question:** `G5000-39`, hai hướng năng lực HPE qua Axis Security và thông báo AI cloud.
- **Kết quả:** cả Flat và GraphRAG đều đạt comprehensiveness `3`, faithfulness `4`, multi-hop `3`.
- **Root cause:** schema allowlist có `ACQUIRED` nhưng không có relation `OFFERS`; vì vậy graph biểu diễn tốt thương vụ Axis Security nhưng thiếu cạnh mô tả năng lực AI cloud. Vector context cũng chưa nêu đủ hai capability một cách trực tiếp.
- **Khắc phục:** mở rộng relation schema có kiểm soát (`OFFERS`, `ENABLES`), tăng extraction coverage cho Golden evidence, rồi chạy regression eval trước khi thay đổi production schema.

### 5. Trade-offs, kiểm soát AI Coding Agent và scale 350 MB

- **Quality/Cost/Latency:** Flat RAG có indexing đơn giản và thường rẻ hơn; GraphRAG chịu chi phí extraction, entity resolution và graph traversal nhưng cải thiện rõ các câu multi-hop. Trong lần chạy này quality tăng 0,6 điểm, latency chỉ tăng 0,029 giây và token còn giảm nhẹ, nhưng không nên suy rộng từ 5 queries.
- **Quyết định từ chối đề xuất agent:** không dùng pairwise cosine `O(N²)` trên toàn bộ entity; thay bằng FAISS candidate search, type blocking và lexical guard. Cũng không gửi toàn bộ CSV 300 MB qua LLM mà giữ scale guard 400 extraction chunks.
- **Scale 350 MB:** bottleneck đầu tiên là LLM extraction/rate limit, sau đó là entity resolution và write throughput. Thiết kế đề xuất: streaming + content hash, durable batch queue, idempotent checkpoints, async workers có backpressure, schema validation, ANN blocking, Neo4j `UNWIND`, partition theo thời gian/domain và theo dõi cost/error rate.

---

## PHẦN 2 — SUY NGẪM & KẾ HOẠCH ĐỒ ÁN

### 1. Mapping bài giảng vào code

| Khái niệm | Module | Hàm/khối code | Quan sát thực tế |
|---|---|---|---|
| Conservative Coreference | M1 | `needs_coref()`, `resolve_coref_batch()`, `run_coref()` | Chỉ gọi LLM cho 178/400 chunks; cache và passthrough giữ pipeline an toàn |
| Schema & Allowlist Guard | M2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS`, `safe_confidence()` | Loại quan hệ ngoài schema; đổi lại có thể giảm recall như G5000-39 |
| Bulk Cypher Ingestion | M2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND` theo batch; 289 nodes/163 edges, provenance lỗi bằng 0 |
| Entity Resolution & Union-Find | M3 | `build_resolution_map()`, `merge_guard()`, `UF` | Affirm được merge; Liverpool/Liverpool FC bị chặn |
| Super-node Degree Cap | M4 | `retrieve_graph_context()`, `recent_edges()` | Có cap 50/node và global cap 250; sample chưa có node degree > 100 |
| LLM-as-a-Judge | M5 | `judge_answer()`, `run_evaluation()`, `comparison_table()` | 5 câu đủ factoid/multi-hop/cross-doc, có rationale và checkpoint |

### 2. Quá trình debugging & bài học

- **Lỗi khó nhất:** model trả JSON hợp lệ nhưng bỏ qua chunk không có phép coreference/quan hệ, trong khi code ban đầu bắt buộc tập `chunk_id` trả về phải bằng tuyệt đối input; điều này làm nhiều batch bị báo lỗi giả. Đồng thời Groq chạm quota `429`, và CSV dùng cột `description` thay vì các tên text phổ biến.
- **Cách xử lý:** mở rộng loader nhận `description`; ưu tiên Golden evidence rows; chuyển workload sang `gpt-4o-mini`; chạy batch song song; chỉ chặn ID lạ/trùng, còn missing item được hiểu là zero-result conservative; coreference có checkpoint cache; mọi lỗi thật vẫn được log và ngăn ingestion.
- **Bài học:** validation production phải phân biệt “không có kết quả” với “output sai contract”; checkpoint và idempotency quan trọng hơn tăng retry; benchmark phải bám evidence thật thay vì câu hỏi mẫu không có đáp án.

### 3. Kế hoạch áp dụng vào đồ án thực tế

- **Đề án:** hệ thống theo dõi hệ sinh thái công nghệ và quan hệ giữa nhà cung cấp, sản phẩm và khách hàng.
- **Lựa chọn kiến trúc:** dùng Hybrid RAG. Flat RAG phục vụ factoid/semantic lookup; GraphRAG dành cho câu hỏi chuỗi cung ứng, đối tác, M&A và timeline nhiều tài liệu.
- **Nodes:** `Company`, `Person`, `Technology`, `Product`, `Event`, `Document`.
- **Relations:** `DEVELOPED`, `USES`, `PARTNERED_WITH`, `ACQUIRED`, `INVESTED_IN`, `FOUNDED`, `WORKED_AT`, `ANNOUNCED`.
- **Entity resolution:** manual aliases cho ticker/vendor lớn; normalize; block theo type/domain; ANN top-k; lexical guard; human-review queue cho vùng similarity không chắc chắn.
- **Super-node:** relevance + temporal filters trước traversal, quota theo relation type, cap per node, global edge/token budget và community summaries cho câu hỏi vĩ mô.

### 4. Tự đánh giá

| Tiêu chí | Điểm (1–5) | Ghi chú |
|---|---:|---|
| Hiểu bài giảng GraphRAG | 5 | Triển khai đủ preprocessing, KG, hybrid retrieval và eval |
| Kiểm soát AI Coding Agent | 5 | Kiểm tra schema, quota, cache và không chấp nhận all-pairs O(N²) |
| Chất lượng knowledge graph | 4 | Provenance đầy đủ; relation schema còn hẹp |
| Phân tích/debug hệ thống | 5 | Truy vết lỗi CSV, API quota và partial JSON response |

---

## Kết luận

Trong sample này, GraphRAG cải thiện trung bình **+0,6 điểm** trên cả ba tiêu chí chất lượng, đặc biệt rõ ở multi-hop, trong khi latency gần tương đương. Kết quả 5 queries chỉ là bằng chứng lab; bước tiếp theo là mở rộng Golden Dataset, audit thêm entity candidates và theo dõi cost/quality theo từng phiên bản extraction schema.
