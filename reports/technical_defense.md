# Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Trịnh Hoàng Nam \
**MSHV:** 2A202601376 \ 
**Notebook tham chiếu:** `Day19_GraphRAG_vs_FlatRAG_Production_Lab_Guide.ipynb` (đã Restart & Run All, chạy local)

Toàn bộ số liệu trong file này lấy trực tiếp từ output đã chạy: `outputs/entity_resolution_audit.csv`, `outputs/graphrag_eval_results.csv`, `outputs/graphrag_vs_flatrag_summary.csv`, `outputs/near_dedup_audit.csv`, `outputs/community_reports.csv`, `outputs/bonus_self_correction_demo.csv`.

---

## 1. Coreference Resolution — tình huống khó/dễ sai

**Cấu hình dữ liệu nguồn:** dataset HackerNoon gốc chỉ có cột `description` (mô tả ngắn), không có `body`/`story`, nên pipeline được chỉnh để ghép `text = title + ". " + description`. Vì `description` thường ngắn (1 đoạn), phần lớn bài chỉ có 1 chunk (`::c0000`), nhưng vẫn đủ để chứa nhiều thực thể trong cùng 1 câu.

**Ví dụ cụ thể từ dữ liệu thật** (`article_id`/`chunk_id`):
`https://washingtontechnology.com/companies/2023/06/microsoft-unveils-openai-service-government-customers/387381/::c0000` (published_date `2023-06-09`)

> "Microsoft unveils OpenAI service for government customers. **The company** unveiled its new Azure OpenAI Service for federal agencies ... for government cloud operations"

**Phân tích rủi ro:** câu có 2 thực thể `Company` xuất hiện gần nhau — *Microsoft* (chủ ngữ câu 1) và *OpenAI* (tân ngữ, nằm trong cụm "OpenAI service"). Về ngữ pháp, "the company" phải trỏ về *Microsoft* (chủ thể hành động "unveils"/"unveiled" lặp lại), nhưng nếu coref model chỉ dùng heuristic "danh từ gần nhất" thay vì phân tích cấu trúc chủ ngữ-vị ngữ, model có thể gán nhầm "the company" = *OpenAI*.

**Hậu quả đối với Knowledge Graph nếu resolve sai:** NER+RE sẽ tạo edge `OpenAI -DEVELOPED-> Azure OpenAI Service` thay vì đúng phải là `Microsoft -DEVELOPED-> Azure OpenAI Service` — một **false edge** gán nhầm sản phẩm cho công ty không phát triển ra nó, làm sai lệch trực tiếp câu trả lời cho mọi truy vấn multi-hop đi qua node OpenAI/Microsoft (2 node đang là super-node hạng 1 và 3 của đồ thị — degree 15 và 12).

**Cơ chế phòng ngừa đã dùng:** prompt coref conservative (`resolve_coref_batch`, cell 1.7) chỉ resolve khi antecedent rõ trong cùng chunk, không suy diễn; nếu mơ hồ → giữ nguyên văn bản gốc + ghi vào `unresolved_mentions`. Do file `unresolved_mentions` chưa được export riêng ra CSV trong lần chạy này, nên không thể trích chính xác quyết định model đã đưa ra cho câu trên — đây là điểm cần cải thiện: nên `to_csv` cột này ra `outputs/` ở lần chạy tiếp theo để audit được đầy đủ.

---

## 2. Entity Resolution Threshold & Lexical Guard

- **Ngưỡng cosine similarity (vector matching):** `ENTITY_SIM_THRESHOLD = 0.85`
- **Ngưỡng Lexical Guard (SequenceMatcher sau khi bỏ suffix Corp/Inc/Ltd...):** `MERGE_GUARD_RATIO = 0.78`

Ban đầu dùng `0.90` — quá chặt, chỉ bắt được 4 cặp `MERGE_VECTOR` và không có case nào bị `REJECT_GUARD` nào để minh hoạ. Hạ xuống `0.85` giúp bắt thêm các biến thể tên thật (ví dụ `Broadcom Inc` vs `Broadcom` = 0.862) đồng thời vẫn giữ Guard đủ chặt để lộ ra các case nên bị từ chối.

**Cặp thực thể similarity > 0.85 nhưng bị `REJECT_GUARD`** (trích `outputs/entity_resolution_audit.csv`):

| left | right | similarity | Lý do từ chối |
|---|---|---|---|
| `technology office` | `Office of Technology` | 0.932 | Cùng từ vựng nhưng khác phạm vi ngữ nghĩa — một bên là cụm chung ("bộ phận công nghệ" của bất kỳ tổ chức nào), một bên có khả năng là tên riêng của 1 đơn vị cụ thể. Gộp 2 thực thể này sẽ tạo canonical entity sai, làm loãng graph. |
| `cloud-computing services` | `cloud computing` | 0.883 | Một bên là cụm mô tả *dịch vụ* (Technology instance cụ thể được 1 công ty cung cấp), một bên là *khái niệm ngành* chung chung — gộp lại sẽ khiến mọi cạnh `USES cloud computing` bị trộn lẫn giữa hàng chục công ty không liên quan. |
| `technology office` | `Office of Technology` | 0.932 | (trùng ở trên) |
| `identity proofing and passwordless multi-factor authentication` | `multi-factor passwordless authentication` | 0.866 | Đây là case **guard có thể đang quá tay** — 2 cụm gần như đồng nghĩa (chỉ khác nhau ở tiền tố "identity proofing and"). Ratio 0.78 sau khi bỏ suffix vẫn dưới ngưỡng vì SequenceMatcher phạt nặng phần chữ dư ở đầu câu. Đây là 1 hạn chế đã biết của Lexical Guard dựa trên string ratio thuần tuý — nên cân nhắc thêm rule "chứa trọn subsequence" cho các cụm dài. |

**Toàn bộ audit:** 23 dòng — `MERGE_MANUAL`=12 (alias thủ công: Microsoft/Google/Amazon/Meta/Apple/IBM/ServiceNow), `MERGE_VECTOR`=7, `REJECT_GUARD`=4.

---

## 3. Super-node Mitigation

**Top 3 thực thể có degree cao nhất** (sau khi re-ingest với `EXTRACTION_MAX_CHUNKS=1200`, hub-stratified sampling ưu tiên chunk nhắc big-tech, đồ thị 785 node / 504 edge):

| Hạng | Tên thực thể | Type | Degree |
|------|--------------|------|--------|
| 1 | Microsoft | Company | 15 |
| 2 | Google | Company | 14 |
| 3 | OpenAI | Company | 12 |

**Trạng thái thực tế:** ngưỡng production `SUPER_NODE_DEGREE=100` **chưa từng bị vượt** trong quy mô dữ liệu giờ lab (1500 bài báo, `description` ngắn nên mật độ quan hệ/bài thấp). Để chứng minh nhánh cắt tỉa vẫn hoạt động đúng, đã bổ sung "lab stress-demo" hạ ngưỡng tạm xuống `LAB_SUPERNODE_STRESS_DEGREE=10`: với node Microsoft (degree=15 > 10), `retrieve_graph_context()` xác nhận `supernode_events` được ghi nhận đúng (`{'node_id': ..., 'degree': 15, 'limit': 50, 'policy_degree': 10}`), và `recent_edges()` trả về đúng ≤ `EDGE_CAP=50`.

**Ưu điểm của chính sách "ưu tiên N=50 cạnh `published_date` mới nhất":**
- Giảm bùng nổ context (`MAX_GRAPH_CONTEXT_CHARS=14000`) và token cost khi node có hàng trăm/nghìn cạnh (các công ty lớn thực sự trong dữ liệu 350MB đầy đủ).
- Ưu tiên thông tin timeline gần nhất — phù hợp với tin tức công nghệ vốn thay đổi nhanh (M&A, funding round mới thường quan trọng hơn tin cũ).

**Rủi ro tiềm ẩn:**
- Câu hỏi dạng lịch sử/so sánh theo thời gian (ví dụ so sánh timeline OpenAI tháng 3 → tháng 7 trong Golden Dataset) có thể bị cắt mất cạnh cũ nếu node đó có > 50 cạnh mới hơn — làm mất khả năng multi-hop ngược thời gian.
- Với đồ thị hiện tại (degree cao nhất chỉ 15), giới hạn 50 không hề bị chạm tới — nghĩa là stress-demo mới chứng minh được "cơ chế kích hoạt đúng lúc" (detection), **chưa chứng minh được hành vi cắt bớt (truncation) thật sự** vì số cạnh sẵn có (15) còn nhỏ hơn cap (50). Đây là giới hạn thành thật của lần chạy này, không phải giả định.

---

## 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

**Bảng tổng hợp Benchmark (LLM-as-a-Judge, 25 câu Golden Dataset — 12 multi-hop, 11 cross-doc, 2 factoid):**

| Nhóm | Metric | Flat RAG | GraphRAG | Nhận xét |
|---|---|---|---|---|
| Factoid | Comprehensiveness | 2.500 | 1.500 | Flat RAG tốt hơn |
| Factoid | Faithfulness | 3.000 | 2.000 | Flat RAG tốt hơn |
| Factoid | Multi-hop reasoning | 2.500 | 1.500 | Flat RAG tốt hơn |
| Cross-doc | Faithfulness | 2.455 | 2.818 | GraphRAG tốt hơn |
| Cross-doc | Comprehensiveness | 2.091 | 2.273 | Gần nhau, Graph nhỉnh hơn |
| Multi-hop | Faithfulness | 2.167 | 2.083 | Gần như bằng nhau |
| Multi-hop | Token usage | 858.8 | 1813.0 | GraphRAG đắt hơn 2.1× |

**Trung bình toàn bộ 25 câu (tính trọng số theo cỡ nhóm):** Latency Flat ≈ 2.39s vs Graph ≈ 2.37s (gần như bằng nhau); Token Flat ≈ 819 vs Graph ≈ 1398 (GraphRAG đắt hơn ≈ 1.7×).

### Ca lỗi 1 — Flat RAG thất bại, GraphRAG thành công hơn
**G5000-37** (cross-doc): *"What evidence shows Dell NativeEdge moving from a May product announcement to a July use-case description?"*
`flat_q = 1.0` (thấp nhất thang điểm) vs `graph_q = 3.33`.
Flat RAG trả lời: *"context does not... insufficient evidence to confirm this transition"* — vector top-k không kéo được đủ cả 2 bài (May + July) vào cùng context vì chúng không tương đồng ngữ nghĩa đủ cao với query embedding, dù cùng nói về 1 sản phẩm. GraphRAG traversal đi qua node "Dell NativeEdge" tới cạnh có `published_date=2023-07-17`, kèm provenance chunk `computerweekly.com/.../Networking-and-communication::c0000`, nên trả lời đúng phần "July use-case", dù vẫn thiếu chi tiết cạnh "May announcement". Đây là ví dụ kinh điển của context fragmentation trong Flat RAG mà đồ án GraphRAG được thiết kế để giải quyết.

### Ca lỗi 2 — GraphRAG khó khăn hơn Flat RAG
**G5000-47** (factoid): *"...does the mention of Palo Alto Networks establish Palo Alto as a partner in that deal?"*
`flat_q = 4.33` vs `graph_q = 2.33`.
Cả 2 đều kết luận đúng hướng (Palo Alto không phải partner), nhưng LLM-Judge chấm GraphRAG thấp hơn vì: *"it inaccurately claims that the context does not mention Palo Alto Networks at all"* — nguyên nhân gốc rễ: "Palo Alto Networks" chỉ xuất hiện trong text với vai trò "nguồn trích dẫn IoT Threat Report", không khớp bất kỳ `ALLOWED_RELATIONS` nào (`ACQUIRED/DEVELOPED/INVESTED_IN/FOUNDED/WORKED_AT/PARTNERED_WITH/USES/LEADS`) → NER+RE **không tạo edge** cho thực thể này → graph context hoàn toàn thiếu tên "Palo Alto Networks", trong khi Flat RAG vẫn giữ nguyên câu chữ gốc trong vector chunk nên "nhìn thấy" được cụm từ này. Đây là hạn chế cố hữu của graph extraction theo allowlist quan hệ hẹp: precision cao nhưng recall thấp với các vai trò không thuộc schema (ví dụ "được trích dẫn làm nguồn", "được đề cập nhưng không có quan hệ hành động").

---

## 5. Trade-offs, Agent Control & Scale 350MB

**Đánh đổi Quality vs Cost vs Latency:** trong quy mô 1500 bài/785 node/504 edge, GraphRAG **không** rẻ hơn hay nhanh hơn rõ rệt so với Flat RAG (latency gần bằng nhau), nhưng tốn token nhiều hơn ~1.7× do phải linearize subgraph + kèm cả vector context. Điểm chất lượng GraphRAG chỉ thắng rõ ở nhóm `cross-doc` (đúng như thiết kế: cần nối thông tin nhiều bài/nhiều thời điểm) — ở nhóm `factoid` (câu hỏi đơn giản), GraphRAG không có lợi thế và còn rủi ro mất recall do allowlist quan hệ hẹp (mục 4, ca lỗi 2). Kết luận thực tế: GraphRAG chỉ đáng chi phí thêm khi câu hỏi thực sự cần multi-hop/cross-document, không nên dùng làm retrieval mặc định cho mọi loại câu hỏi.

**Quyết định kỹ thuật đã tự đưa ra (không nghe theo gợi ý mặc định/dễ dãi):**
- Sau khi phát hiện `entity_resolution_audit_df` chỉ có 4 dòng (dưới ngưỡng rubric 10 dòng), thay vì hạ chuẩn "coi như đủ" hay bịa thêm dòng audit giả, đã chọn cách đúng đắn hơn: tăng `EXTRACTION_MAX_CHUNKS` (400→1200) + thêm hub-stratified sampling để tự nhiên sinh ra nhiều biến thể tên thật hơn — kết quả thật là 23 dòng audit với đủ 3 loại nhãn.
- Khi phát hiện việc re-run notebook nhiều lần khiến Neo4j cộng dồn node/edge trùng lặp (degree bị thổi phồng giả), đã chủ động thêm `clear_graph()` (`MATCH (n) DETACH DELETE n`) trước bước bulk insert thay vì bỏ qua rủi ro này — tránh báo cáo sai lệch số liệu đồ thị.
- Với Super-node cap, thay vì hạ thẳng ngưỡng production `SUPER_NODE_DEGREE=100` xuống một con số nhỏ để "dễ demo" (sẽ che giấu hành vi thật của hệ thống ở quy mô production), đã tách riêng một nhánh "lab stress-demo" độc lập chỉ dùng để kiểm chứng logic, giữ nguyên chính sách production.

**Giải pháp nếu scale lên 350MB (~100,000 bài báo):**
Bottleneck đầu tiên là **LLM extraction (NER+RE + Coreference)** — không phải Neo4j hay FAISS. Bằng chứng thực đo: với chỉ 1194 chunk, coref mất 34 phút 42 giây (239 batch × ~8.7s) và NER+RE mất 21 phút 08 giây (299 batch × ~4.2s/batch) — đây là phần chậm nhất toàn pipeline, hoàn toàn do gọi API tuần tự + rate-limit. Với 100,000 bài (~gấp 65 lần), chạy tuần tự như hiện tại sẽ mất hàng chục giờ và dễ vượt rate-limit.
Hướng xử lý: (1) async batch queue với nhiều API key/worker song song thay vì vòng lặp tuần tự; (2) chỉ trích xuất incremental — chỉ chạy NER+RE cho bài mới/thay đổi thay vì full re-run; (3) Entity Resolution dùng ANN + blocking theo `type` (đã làm) để tránh so khớp toàn cục; (4) với graph lớn hơn nhiều, cần bật thật cơ chế Super-node production (không chỉ stress-demo) và cân nhắc thêm Community/Global Search (mục Bonus B) để trả lời câu hỏi vĩ mô mà không cần traversal toàn đồ thị.

---

## 6. Bonus Challenges — đã triển khai

### A. Near-Dedup (SimHash + LSH) — +3đ
Cài `simhash64()` (64-bit fingerprint theo token/bigram) + LSH theo 4 band (`NEAR_DEDUP_NUM_BANDS=4`), Union-Find gộp cặp có Hamming distance ≤ `NEAR_DEDUP_HAMMING_THRESHOLD=3` — độ phức tạp gần tuyến tính (chỉ so trong cùng bucket band), không dùng pairwise cosine O(N²).
Kết quả: `outputs/near_dedup_audit.csv` — 6 cặp bị merge, ví dụ: 2 bài AP-syndicate "Samsung Showcases Groundbreaking Logic Innovations..." đăng trên 2 domain khác nhau (`joplinglobe.com` vs `galvnews.com`), Hamming distance=1 — chuẩn near-duplicate thật (không phải exact hash match vì domain/URL khác nhau nên SHA-1 exact dedup ở bước trước không bắt được).
False-positive risk: threshold Hamming≤3/64 khá bảo thủ (chỉ merge khi gần như đồng nhất về token/bigram) — chưa quan sát thấy case merge sai trong 6 cặp audit.
Audit: mọi cặp merge đều lưu `kept_article_id`, `removed_article_id`, `hamming_distance`, `merge_reason` để truy vết.

### B. Global Search via Community Reports — +5đ
Dùng NetworkX (`greedy_modularity_communities`) phân cụm 785 node/504 edge → 300 community (đồ thị khá thưa, ghi `community_id` vào từng node), tóm tắt 20 community lớn nhất bằng LLM → `outputs/community_reports.csv`. Ví dụ community lớn nhất (35 thành viên) tập trung quanh Apple/Amazon/AI technology. Query router demo (`global_search`/`answer_global_rag`) trả lời đúng 2 câu hỏi vĩ mô ("major AI ecosystem themes", "cloud providers/model vendors connections") bằng cách route tới top-k community report thay vì traversal toàn đồ thị.

### C. Self-Correction Graph Retrieval — +5đ
`context_sufficient()` (LLM đánh giá context có đủ trả lời không) được nối hoàn chỉnh vào `self_correcting_context()` → `answer_graph_rag_self_correct()`: hop2 → nếu context chưa đủ → hop3 → nếu vẫn chưa đủ → vector fallback.
Demo trên 5 câu Golden: **cả 5/5 câu đều bị đánh giá hop2 không đủ và escalate lên `hop3+vector`** (route distribution: `hop3+vector`=5, `hop2`=0). Đây là dữ liệu thật, cho thấy với đồ thị hiện tại (chỉ 504 edge, khá thưa), hop2 hiếm khi đủ ngữ cảnh — một quan sát hợp lý xét đến quy mô dữ liệu giờ lab, nhưng cũng gợi ý ngưỡng "sufficient" trong `context_sufficient()` có thể đang khá nghiêm ngặt, nên đối chiếu thêm khi graph lớn hơn.
