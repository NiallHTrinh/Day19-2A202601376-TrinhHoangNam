# Suy Ngẫm Cá Nhân & Kế Hoạch Đồ Án — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Trịnh Hoàng Nam \
**MSHV:** 2A202601376

---

## 1. Mapping Bài giảng vào Code

| Khái niệm trong bài giảng | Module | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|---|---|---|---|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()`, `run_coref()` (cell 1.7) | Prompt yêu cầu chỉ resolve khi antecedent rõ trong cùng chunk; đã kiểm chứng bằng ví dụ Microsoft/OpenAI trong `technical_defense.md` §1 — cần export `unresolved_mentions` ra CSV riêng để audit đầy đủ hơn ở lần sau. |
| **Near-Dedup (SimHash+LSH)** | Module 1 (Bonus A) | `simhash64()`, `near_dedup_simhash_lsh()` (cell 1.5) | Tự thêm ngoài yêu cầu bắt buộc; loại được 6 cặp bài near-duplicate syndicate mà exact-hash bỏ sót. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` (cell 2.1) | Allowlist hẹp giúp graph sạch (0 cạnh thiếu provenance) nhưng cũng là nguyên nhân gốc của ca lỗi 2 trong `failure_analysis.md` (thực thể không có quan hệ hợp lệ bị loại hoàn toàn). |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()`, `clear_graph()` (cell 2.3) | Dùng đúng `UNWIND $rows AS row`; tự thêm `clear_graph()` để tránh cộng dồn degree giả khi re-run nhiều lần. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, class `UF` (cell 2.2) | Threshold ban đầu 0.90 quá chặt (chỉ 4 audit row) → hạ xuống 0.85 + `MERGE_GUARD_RATIO=0.78` → 23 audit row, đủ 3 loại nhãn. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()`, `recent_edges()`, `test_supernode_policy()` (cell 3.3, 5.1) | Đồ thị lab-scale (785 node/504 edge, max degree=15) chưa tự nhiên sinh ra super-node thật (ngưỡng production=100) → tự thêm nhánh "lab stress-demo" (`LAB_SUPERNODE_STRESS_DEGREE=10`) để verify logic mà không sửa chính sách production. |
| **Global Search / Community Detection** | Module 5 (Bonus B) | `build_communities()`, `build_community_reports()`, `global_search()`, `answer_global_rag()` (Bonus cell) | 300 community từ NetworkX `greedy_modularity_communities`; 20 report lớn nhất được LLM tóm tắt; demo trả lời đúng 2 câu hỏi vĩ mô. |
| **Self-Correction Retrieval** | Module 5 (Bonus C) | `context_sufficient()`, `self_correcting_context()`, `answer_graph_rag_self_correct()` (Bonus cell) | Nối trọn vẹn hop2→hop3→vector fallback; demo 5 câu cho thấy 100% escalate lên `hop3+vector` — gợi ý hop2 hiếm khi đủ với graph thưa hiện tại. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()`, `run_evaluation()`, `comparison_table()` (cell 4.2–4.4) | Chạy đủ 25 câu Golden Dataset (12 multi-hop, 11 cross-doc, 2 factoid), xuất `graphrag_eval_results.csv` + `graphrag_vs_flatrag_summary.csv`. |

---

## 2. Quá trình Debugging & Bài học

**Lỗi kỹ thuật phức tạp nhất gặp phải:** không phải 1 lỗi runtime đơn lẻ, mà là một chuỗi vấn đề **số liệu không đủ để chứng minh các cơ chế bắt buộc** dù code không hề crash — khó phát hiện hơn lỗi cú pháp vì notebook vẫn chạy "OK" bình thường:

1. **Entity Resolution audit chỉ có 4 dòng** (rubric yêu cầu ≥10, đủ 3 loại nhãn) — nguyên nhân là `EXTRACTION_MAX_CHUNKS=400` quá ít nên không đủ số lần một thực thể xuất hiện dưới nhiều biến thể tên để vector matching bắt được, và threshold `0.90` quá chặt nên không lộ ra case nào bị `REJECT_GUARD`.
2. **Super-node cap không bao giờ được kích hoạt thật** — vì đồ thị quá thưa (max degree chỉ 13–15, ngưỡng production là 100) nên nhánh code cắt tỉa 50 cạnh chưa từng chạy qua dù đã viết đúng.
3. **Re-run notebook nhiều lần cộng dồn dữ liệu vào Neo4j** — mỗi lần bulk insert lại mà không xoá dữ liệu cũ khiến degree/count bị thổi phồng giả, số liệu báo cáo sẽ sai nếu không phát hiện kịp.

**Cách xử lý:** thay vì "vá tạm" (hạ chuẩn rubric, bịa audit row, hoặc hạ thẳng ngưỡng production để dễ demo), đã chọn hướng khắc phục đúng gốc: tăng `EXTRACTION_MAX_CHUNKS` (400→1200) kèm hub-stratified sampling để tăng mật độ đồ thị một cách tự nhiên; hạ `ENTITY_SIM_THRESHOLD` xuống 0.85 một cách có chủ đích (không phải ngẫu nhiên); thêm `clear_graph()` trước mỗi lần re-ingest; và tách riêng nhánh "lab stress-demo" độc lập với chính sách production để vừa chứng minh được logic vừa không che giấu hành vi thật ở quy mô lớn. Bài học lớn nhất: **một notebook chạy không lỗi không đồng nghĩa với việc pipeline đã chứng minh được đúng các failure-mode mà đề bài yêu cầu** — cần chủ động kiểm tra số liệu đầu ra (không chỉ execution status) đối chiếu với từng tiêu chí rubric trước khi kết luận đã "xong".

---

## 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)

- **Tên đồ án / Dự án:** Hệ thống RAG tra cứu & hỏi-đáp Văn bản Quy định/Quy chế Nội bộ Trường Đại học (quy chế đào tạo, quy định học phí, học bổng, kỷ luật, công tác sinh viên...).

- **Đặc thù bài toán & Lý do chọn giải pháp:**
  Văn bản quy chế có 3 đặc điểm khớp đúng với nhóm câu hỏi mà lab này đã chứng minh GraphRAG có lợi thế (`cross-doc`, `multi-hop`), nên **GraphRAG/Hybrid RAG là lựa chọn phù hợp**, không chỉ dùng Flat RAG thuần:
  1. **Tham chiếu chéo dày đặc giữa các điều khoản** — ví dụ "Điều 15 quy định về xử lý học vụ, dẫn chiếu Điều 8 về điều kiện học tiếp" — câu hỏi kiểu này bản chất là multi-hop, giống pattern ca lỗi G5000-37 (Dell NativeEdge) trong lab: Flat RAG dễ chỉ lấy đúng 1 điều khoản do similarity cao nhất, bỏ sót điều khoản liên đới có similarity thấp hơn nhưng lại là mắt xích bắt buộc.
  2. **Văn bản có tính lịch sử/hiệu lực (temporal validity)** — quy chế thường được sửa đổi, thay thế qua nhiều quyết định theo thời gian (ví dụ Quy chế 2021 bị thay thế một phần bởi Quyết định sửa đổi 2023). Câu hỏi "quy định hiện hành về X là gì" bắt buộc phải biết văn bản nào còn hiệu lực — đúng vai trò của thuộc tính `published_date`/provenance trong lab, chỉ khác là ở đây cần thêm `effective_date`/`superseded_by` tường minh thay vì chỉ ưu tiên "mới nhất" một cách ngầm định.
  3. Ngược lại, nhiều câu hỏi tra cứu đơn lẻ ("Điều 5 quy định gì?", "Mức học phí ngành X là bao nhiêu?") thuộc nhóm `factoid` — giống ca lỗi G5000-47 trong lab, nơi Flat RAG thắng vì không cần suy luận nối quan hệ. Vì vậy hệ thống nên dùng **kiến trúc Hybrid**: route câu hỏi factoid đơn giản sang Flat RAG (nhanh, rẻ, đủ chính xác), chỉ kích hoạt Graph traversal khi phát hiện câu hỏi cần đối chiếu ≥2 điều khoản/văn bản — tránh lặp lại nhược điểm quan sát được ở lab (GraphRAG tốn token gấp ~1.7× mà không có lợi ở nhóm factoid).

- **Cấu trúc Node & Relation dự kiến:**
  - **Nodes:** `Document` (Quy chế/Quyết định/Thông tư, có thuộc tính `issue_date`, `effective_date`, `status`: hiệu lực/hết hiệu lực), `Chapter` (Chương), `Article` (Điều), `Clause` (Khoản/Điểm), `Actor` (đối tượng áp dụng: Sinh viên, Giảng viên, Phòng Đào tạo...), `Concept` (khái niệm được định nghĩa tường minh trong văn bản, ví dụ "học bổng khuyến khích học tập", "cảnh báo học vụ").
  - **Relations:** `CONTAINS` (Document→Chapter→Article→Clause, quan hệ cấu trúc phân cấp), `REFERENCES` (Điều X tham chiếu/dẫn chiếu Điều Y), `AMENDS` / `SUPERSEDES` (văn bản mới sửa đổi/thay thế văn bản cũ — bắt buộc có `effective_date` làm provenance, tương tự `published_date` trong lab), `APPLIES_TO` (Điều áp dụng cho Actor nào), `DEFINES` (Điều định nghĩa Concept nào). Giữ allowlist **hẹp và tường minh** như lab (ưu tiên precision), vì văn bản quy phạm đòi hỏi độ chính xác pháp lý cao hơn nhiều so với tin tức — một false edge ở đây (ví dụ gán nhầm Điều nào sửa đổi Điều nào) có thể dẫn đến tư vấn sai quy định cho sinh viên.

- **Chiến lược xử lý Super-node & Entity Resolution:**
  - **Super-node dự kiến:** (1) chính `Document` gốc — vì mọi Điều/Khoản đều `CONTAINS` từ nó, degree sẽ rất cao một cách tự nhiên nhưng **không mang giá trị truy hồi** (chỉ là quan hệ cấu trúc) → nên **loại quan hệ `CONTAINS` ra khỏi việc tính degree cap**, chỉ áp dụng `SUPER_NODE_DEGREE`/`EDGE_CAP` cho các quan hệ ngữ nghĩa thật (`REFERENCES`, `AMENDS`, `APPLIES_TO`); (2) các Điều "định nghĩa/giải thích từ ngữ" (thường là Điều 2 trong đa số quy chế) — bị rất nhiều Điều khác `REFERENCES` tới, đúng là super-node thật cần cap giống Microsoft/Google trong lab, nhưng chính sách ưu tiên "N cạnh mới nhất" (`published_date`) của lab **không phù hợp trực tiếp** ở đây — nên đổi tiêu chí ưu tiên sang "cạnh thuộc văn bản đang còn hiệu lực (`status=active`) trước, sau đó mới theo `effective_date` gần nhất", vì với quy chế thì "còn hiệu lực" quan trọng hơn "mới nhất".
  - **Entity Resolution:** tên văn bản chính thức thường có nhiều cách viết tắt cố định trong nội bộ trường (ví dụ "Quy chế đào tạo trình độ đại học 2021" ↔ "QCĐT 2021" ↔ "Quy chế 26"), đây là biến thể **có quy tắc, không ngẫu nhiên như tên công ty trong tin tức** → nên ưu tiên tầng `Manual Alias Map` (build từ danh mục văn bản chính thức của trường) làm tuyến chính, tầng Vector ANN + Lexical Guard chỉ dùng làm fallback bắt lỗi chính tả/OCR khi số hoá văn bản PDF. Coreference resolution ít quan trọng hơn so với lab (văn bản pháp quy hiếm dùng đại từ mơ hồ, thường lặp lại tên đầy đủ theo đúng văn phong hành chính), nên có thể giảm ưu tiên đầu tư so với phần Entity Resolution & temporal validity.

---

## 🎯 TỰ ĐÁNH GIÁ

> Điểm gợi ý dựa trên bằng chứng thực tế quan sát được trong quá trình làm việc — bạn nên tự điều chỉnh theo cảm nhận thật của mình trước khi nộp.

| Tiêu chí | Điểm tự chấm (1–5, gợi ý) | Ghi chú |
|---|---|---|
| Mức độ hiểu bài giảng GraphRAG | 4 | Hiểu rõ và giải thích được đánh đổi giữa 2 kiến trúc qua 2 ca lỗi cụ thể, không chỉ dừng ở mức chạy code. |
| Khả năng kiểm soát AI Coding Agent | 4 | Tự phát hiện và chọn hướng khắc phục đúng gốc thay vì hạ chuẩn để "qua rubric" (mục 2 ở trên). |
| Chất lượng đồ thị tri thức xây dựng | 3 | Provenance đạt 100%, nhưng đồ thị vẫn khá thưa (504 edge) so với quy mô lý tưởng để thể hiện đầy đủ super-node thật. |
| Khả năng phân tích và debug hệ thống | 4 | Truy được root-cause cụ thể cho cả lỗi định lượng (audit rows) lẫn lỗi định tính (allowlist recall). |
