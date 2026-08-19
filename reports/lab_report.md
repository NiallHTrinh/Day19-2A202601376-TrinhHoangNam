# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Trịnh Hoàng Nam \
**MSHV:** 2A202601376 \
**Ngày thực hiện:** 19/08/2026

> File này là bản tổng hợp theo đúng cấu trúc gốc (README/templates). Nội dung chi tiết đầy đủ, có trích dẫn số liệu/bằng chứng cụ thể nằm ở 3 file riêng theo yêu cầu RUBRIC.md: [`technical_defense.md`](technical_defense.md), [`failure_analysis.md`](failure_analysis.md), [`reflection_TrinhHoangNam.md`](reflection_TrinhHoangNam.md).

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
Ví dụ thật từ dữ liệu: chunk `...microsoft-unveils-openai-service-government-customers/387381/::c0000` — câu *"Microsoft unveils OpenAI service... The company unveiled its new Azure OpenAI Service..."* có 2 công ty (Microsoft, OpenAI) đứng gần nhau; nếu coref chỉ dùng heuristic "danh từ gần nhất" thay vì cấu trúc chủ ngữ-vị ngữ, "the company" có thể bị gán nhầm sang OpenAI. Hậu quả nếu sai: NER+RE sẽ tạo false edge `OpenAI -DEVELOPED-> Azure OpenAI Service`. Chi tiết đầy đủ: `technical_defense.md` §1.

### 2. Entity Resolution Threshold & Lexical Guard
Ngưỡng cosine similarity: `threshold = 0.85`, Lexical Guard `MERGE_GUARD_RATIO = 0.78`. Cặp similarity > 0.85 bị chặn: `technology office` vs `Office of Technology` (sim 0.932, `REJECT_GUARD`) — cùng từ vựng nhưng khác phạm vi thực thể. Toàn bộ audit: 23 dòng (`MERGE_MANUAL`=12, `MERGE_VECTOR`=7, `REJECT_GUARD`=4). Chi tiết: `technical_defense.md` §2.

### 3. Đồ thị & Super-node Mitigation

| Hạng | Tên thực thể | Loại | Degree |
|------|--------------|------|--------|
| 1 | Microsoft | Company | 15 |
| 2 | Google | Company | 14 |
| 3 | OpenAI | Company | 12 |

Ưu điểm: giảm bùng nổ context, ưu tiên tin cập nhật. Rủi ro: có thể cắt mất cạnh cũ cần cho câu hỏi lịch sử. Ở quy mô lab, chưa node nào vượt ngưỡng production (100) — đã chứng minh cơ chế bằng lab stress-demo (threshold=10). Chi tiết: `technical_defense.md` §3.

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

| Tiêu chí đánh giá | Flat RAG | GraphRAG | Δ | Nhận xét |
|---|---|---|---|---|
| Comprehensiveness (TB 25 câu) | 2.00 | 2.00 | 0.00 | Bằng nhau |
| Faithfulness (TB 25 câu) | 2.36 | 2.40 | +0.04 | Gần nhau, Graph thắng riêng nhóm cross-doc |
| Multi-hop reasoning (TB 25 câu) | 1.96 | 1.96 | 0.00 | Bằng nhau |
| Latency trung bình (s) | 2.393 | 2.366 | -0.027 | Gần như bằng nhau |
| Token usage trung bình | 818.4 | 1398.2 | +579.8 | GraphRAG đắt hơn ~1.7× |

Ca lỗi Flat RAG thất bại/GraphRAG thành công hơn: **G5000-37** (Dell NativeEdge May→July, cross-doc), `flat_q=1.0` vs `graph_q=3.33`. Ca lỗi GraphRAG khó khăn hơn: **G5000-47** (Palo Alto Networks, factoid), `flat_q=4.33` vs `graph_q=2.33` — do allowlist quan hệ hẹp khiến node bị loại khỏi graph. Root-cause đầy đủ: `failure_analysis.md`.

### 5. Trade-offs, Agent Control & Scale 350MB
GraphRAG chỉ đáng chi phí thêm khi câu hỏi cần multi-hop/cross-document thật sự (nhóm cross-doc), không nên dùng mặc định cho mọi câu hỏi (nhóm factoid Flat RAG thắng rõ). Quyết định kỹ thuật tự đưa ra thay vì hạ chuẩn: tăng `EXTRACTION_MAX_CHUNKS`, thêm `clear_graph()` trước re-ingest, tách riêng "lab stress-demo" khỏi chính sách production. Bottleneck khi scale 350MB: LLM extraction (NER+RE + coref) — đo thực tế mất ~34 phút coref + ~21 phút NER+RE chỉ với 1194 chunk. Chi tiết: `technical_defense.md` §5.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

Xem đầy đủ bảng mapping 5 module ↔ hàm code, bài học debugging thật (entity audit thiếu dòng → tăng extraction scale; super-node chưa kích hoạt → thêm stress-demo; Neo4j cộng dồn dữ liệu → thêm `clear_graph()`), và kế hoạch đồ án cá nhân tại [`reflection_TrinhHoangNam.md`](reflection_TrinhHoangNam.md).

---

## 🎯 TỰ ĐÁNH GIÁ

| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | 4 | Xem `reflection_TrinhHoangNam.md` |
| Khả năng kiểm soát AI Coding Agent | 4 | |
| Chất lượng đồ thị tri thức xây dựng | 3 | Provenance 100% nhưng graph còn thưa (504 edge) |
| Khả năng phân tích và debug hệ thống | 4 | |
