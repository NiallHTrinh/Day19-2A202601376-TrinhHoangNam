# Failure Analysis — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Trịnh Hoàng Nam \
**MSHV:** 2A202601376 \
**Nguồn số liệu:** `outputs/graphrag_eval_results.csv`, `outputs/graphrag_vs_flatrag_summary.csv`, `outputs/entity_resolution_audit.csv` (kết quả chạy thật từ `Day19_GraphRAG_vs_FlatRAG_Production_Lab_Guide.ipynb`)

Phân tích dưới đây theo quy trình root-cause: **Triệu chứng → Bằng chứng → Nguyên nhân gốc rễ → Đề xuất khắc phục**, cho 2 ca lỗi điển hình trích từ 25 câu Golden Evaluation thật đã chạy.

---

## Ca lỗi 1 — Flat RAG thất bại, GraphRAG thành công hơn

**Câu hỏi:** G5000-37 (nhóm `cross-doc`) — *"What evidence shows Dell NativeEdge moving from a May product announcement to a July use-case description?"*

**Reference answer:** bài Dell Technologies World tháng 5 công bố ra mắt NativeEdge; bài networking tháng 7 mô tả NativeEdge như một platform giúp triển khai edge application/infrastructure dễ dàng hơn qua validated design cho từng ngành.

### Triệu chứng
- `flat_comprehensiveness=1, flat_faithfulness=1, flat_multi_hop_reasoning=1` → điểm sàn tuyệt đối (flat_q = 1.0/5).
- `graph_comprehensiveness=3, graph_faithfulness=4, graph_multi_hop_reasoning=3` → graph_q = 3.33/5, không hoàn hảo nhưng vượt trội rõ rệt.

### Bằng chứng
Flat RAG trả lời: *"...is not explicitly provided in the supplied context... insufficient evidence to confirm this transition."*
Judge rationale (Flat): *"The candidate fails to provide any evidence or details regarding the transition... incorrectly states that there is insufficient evidence, despite the reference clearly outlining the timeline."*

GraphRAG trả lời: *"...found in the context of the July 17, 2023, article... [chunk_id=https://www.computerweekly.com/resources/Networking-and-communication::c0000]... by July, Dell was focusing on practical applications... following an earlier announcement in May. However, the specific details of the May announcement are not provided..."*

### Nguyên nhân gốc rễ
Flat RAG dùng top-k=6 semantic similarity trên toàn bộ `chunks_df`. Bài tháng 5 (product announcement) và bài tháng 7 (use-case) có nội dung khác góc nhìn (một bên là thông cáo ra mắt, một bên là bài phân tích ứng dụng) nên **embedding similarity giữa 2 chunk này với câu hỏi không đủ gần nhau để cả 2 cùng lọt top-k** — đây chính là hiện tượng "context fragmentation" mà kiến trúc GraphRAG được thiết kế để giải quyết: GraphRAG không phụ thuộc vào similarity trực tiếp giữa câu hỏi và từng chunk rời rạc, mà đi qua entity `Dell NativeEdge` làm cầu nối, nên vẫn traversal được tới cạnh có `published_date=2023-07-17` kèm provenance rõ ràng.
Tuy vậy GraphRAG cũng chưa hoàn hảo — cạnh "May announcement" không xuất hiện trong context, cho thấy hoặc (a) NER+RE không trích được quan hệ nào từ bài tháng 5 (có thể do câu văn không khớp `ALLOWED_RELATIONS`), hoặc (b) BFS/seed matching không đi tới đúng node/cạnh đó trong 2-hop.

### Đề xuất khắc phục
- Với Flat RAG: tăng top-k hoặc thêm re-ranking theo entity overlap (không chỉ embedding similarity thuần) để tăng khả năng gom đủ chunk đa góc nhìn về cùng 1 thực thể.
- Với GraphRAG: kiểm tra lại tại sao cạnh "May announcement" không xuất hiện — có thể do câu văn gốc dùng động từ mô tả (ví dụ "unveiled", "released") không map rõ vào `ALLOWED_RELATIONS` hiện có (`DEVELOPED` là gần nhất nhưng NER+RE có thể đã bỏ sót); nên xem xét mở rộng relation `LAUNCHED`/`ANNOUNCED` nếu triển khai tiếp.

---

## Ca lỗi 2 — GraphRAG khó khăn hơn Flat RAG

**Câu hỏi:** G5000-47 (nhóm `factoid`) — *"In the Keysight–Synopsys IoT cybersecurity record, does the mention of Palo Alto Networks establish Palo Alto as a partner in that deal?"*

**Reference answer:** Không. Title chỉ xác định Keysight và Synopsys là đối tác; Palo Alto Networks chỉ xuất hiện với vai trò nguồn của một IoT Threat Report được trích dẫn trong description, không phải bên tham gia deal.

### Triệu chứng
- `flat_q = 4.33/5` (comprehensiveness=4, faithfulness=5, multi_hop=4) — gần như hoàn hảo.
- `graph_q = 2.33/5` (comprehensiveness=2, faithfulness=3, multi_hop=2) — thấp hơn hẳn dù kết luận cuối cùng cùng hướng đúng (Palo Alto không phải partner).

### Bằng chứng
Judge rationale (Graph): *"...it inaccurately claims that the context does not mention Palo Alto Networks at all, while the reference indicates that it is mentioned as a source of an IoT Threat Report. This oversight affects the faithfulness and multi-hop reasoning accuracy."*
GraphRAG trả lời: *"The context provided does not mention Palo Alto Networks in relation to the Keysight–Synopsys IoT cybersecurity record."* — câu trả lời **đúng kết luận nhưng sai lý do**: model tuyên bố "không thấy Palo Alto Networks trong context" trong khi đúng ra Palo Alto Networks có xuất hiện trong văn bản gốc, chỉ là graph context không mang thông tin đó.

### Nguyên nhân gốc rễ
Đây là hạn chế cấu trúc của graph extraction theo **allowlist quan hệ hẹp** (`ACQUIRED, DEVELOPED, INVESTED_IN, FOUNDED, WORKED_AT, PARTNERED_WITH, USES, LEADS`). "Palo Alto Networks" trong văn bản gốc chỉ đóng vai trò *nguồn trích dẫn* (nguồn của 1 threat report), không phải chủ thể của bất kỳ quan hệ nào trong 8 loại được cho phép → NER+RE **không tạo edge nào** cho thực thể này → node "Palo Alto Networks" hoàn toàn vắng mặt trong Neo4j graph → khi GraphRAG linearize subgraph để trả lời, nó không có gì để trích dẫn, dẫn tới phát biểu sai "context không đề cập đến Palo Alto Networks".
Ngược lại, Flat RAG lấy nguyên văn chunk text (không qua bước lọc quan hệ), nên câu chữ gốc "Palo Alto Networks" vẫn còn nguyên trong context vector — giúp Flat RAG trả lời đầy đủ và trung thực hơn dù đây chính là loại câu hỏi (factoid, xác nhận quan hệ) đáng lẽ GraphRAG phải mạnh.

### Đề xuất khắc phục
- Đây là đánh đổi cố hữu giữa **precision** (allowlist hẹp giúp graph sạch, ít false edge) và **recall** (bỏ sót các vai trò không thuộc allowlist như "được trích dẫn làm nguồn", "được đề cập nhưng không hành động"). Không nên mở allowlist tuỳ tiện (dễ làm giảm chất lượng edge như phân tích ở mục Entity Resolution), mà nên bổ sung một node type/relation riêng cho "MENTIONED_AS_SOURCE" hoặc lưu thêm 1 danh sách "mentioned_entities" (không có quan hệ, chỉ để tra cứu) song song với graph triples chính — giúp GraphRAG vẫn "nhìn thấy" các thực thể được nhắc tới dù không có edge, tránh phát biểu sai "không có trong context".
- Về mặt hệ thống: với câu hỏi factoid đơn giản (xác nhận/phủ định 1 quan hệ), nên ưu tiên route sang Flat RAG hoặc Hybrid có trọng số vector cao hơn, thay vì luôn chạy GraphRAG đầy đủ — khớp với kết luận ở mục 4/5 của `technical_defense.md`: GraphRAG chỉ nên dùng khi câu hỏi thực sự cần multi-hop/cross-document.

---

## Tổng kết Root-cause chung

| | Ca 1 (G5000-37) | Ca 2 (G5000-47) |
|---|---|---|
| Bên thua | Flat RAG | GraphRAG |
| Nguyên nhân gốc | Vector similarity không gom đủ 2 chunk khác góc nhìn cùng 1 thực thể | Allowlist quan hệ hẹp khiến 1 thực thể hợp lệ bị loại hoàn toàn khỏi graph |
| Bản chất | Giới hạn của "tìm kiếm ngữ nghĩa phẳng" khi thông tin nằm rời rạc theo thời gian | Giới hạn của "trích xuất có cấu trúc" khi vai trò thực thể không khớp schema định sẵn |
| Bài học | 2 kiến trúc bù trừ lẫn nhau theo đúng loại câu hỏi (cross-doc → Graph thắng; factoid xác nhận quan hệ đơn giản → Flat thắng) — củng cố lý do thiết kế Hybrid Context (kết hợp cả `=== GRAPH ===` và `=== VECTOR ===`) thay vì chỉ dùng 1 trong 2 |
