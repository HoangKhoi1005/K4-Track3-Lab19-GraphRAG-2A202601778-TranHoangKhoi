# Báo cáo Lab 19 — Production GraphRAG vs Flat RAG

**Học viên:** Trần Hoàng Khôi
**Khóa học:** AICB-K34 · Track 3: GraphRAG
**Ngày thực hiện:** 19/08/2026

## Kết quả thực thi

- Download 300MB HackerNoon: 514,417 rows.
- Exact dedup: 245,324 → 212,212 bài; sample chạy pipeline: 1,500 chunks.
- Coreference thành công trên 400 chunks theo conservative policy.
- Neo4j: 89 Entity nodes, 55 edges, 0 edge thiếu `source_chunk_id` hoặc `published_date`.
- Entity-resolution audit: 130 rows.
- FAISS Flat index: 1,500 vectors.
- Benchmark: 5 Golden questions, đủ factoid/multi-hop/cross-doc; đã export hai CSV tại `outputs/`.

## 1. Thuyết minh kỹ thuật và failure analysis

Coreference chỉ thay đại từ khi antecedent rõ trong cùng chunk. Các mention như `we`, `our users'`, `These` được giữ unresolved để tránh false edge. Triple extraction dùng node allowlist Company/Person/Technology và relation allowlist; mỗi edge giữ evidence, confidence, date và chunk nguồn.

Entity Resolution dùng cosine threshold 0.90, lexical guard 0.72, manual aliases và Union-Find. Audit phát hiện false merge `Sonos Era 100`/`Sonos Era 300` (0.903168); nguyên nhân là guard chưa kiểm tra numeric model token. Cải tiến đề xuất là reject khi numeric token khác nhau.

Top degree là 1Kosmos (4), OnePlus (3), Worldcoin (3), nên sample chưa kích hoạt super-node cap. Tuy nhiên retrieval triển khai policy degree >100 → lấy tối đa 50 edge mới nhất, global cap 250 edge và graph context cap 14,000 ký tự.

Test câu hỏi "Which company partnered with Transact?" cho cả Flat RAG và GraphRAG câu trả lời Talkiatry; GraphRAG có provenance `chunk_id=55a09cbc43c41ffb2dd9::c0000`.

Benchmark Golden cũ dùng `first5000` bị lệch với index/graph random 1,500 bài nên chỉ được xem là failure data-alignment, không dùng làm kết luận cuối. Bộ chuẩn `data/golden_dataset.csv` đã được thay bằng 5 câu aligned (1 factoid, 2 multi-hop, 2 cross-doc); 5/5 `chunk_id` evidence đã được kiểm tra có mặt trong corpus hiện tại.

Evaluation aligned đã chạy đủ 5/5 câu và export CSV. Với multi-hop, Flat RAG và GraphRAG đều đạt 5/5 trên ba thang đo. Với cross-doc, GraphRAG đạt 5/5 về comprehensiveness, faithfulness và multi-hop reasoning, trong khi Flat RAG là 1/5 vì không gom được hai evidence chunks của 1Kosmos; đổi lại GraphRAG chậm hơn (6.031s so với 5.201s) và dùng nhiều token hơn (957.5 so với 849.5). Với factoid Transact, Flat RAG đạt 5/5 còn GraphRAG là 1/5: đây là ca graph seed/retrieval không lấy được edge dù edge tồn tại. Như vậy báo cáo có cả ca Flat thất bại/Graph thành công (cross-doc) và ca Graph khó khăn (factoid).

Coreference/NER từng gặp Groq model deprecation và rate limit. Khắc phục là đổi model đang hỗ trợ, test request nhỏ trước batch, checkpoint, pacing và retry. Với scale 350MB, cần worker queue, ANN partition/HNSW, async extraction, batch UNWIND và community partitioning.

## 2. Reflection và Action Plan

Pipeline mapping trực tiếp: `run_coref` (M1), extraction allowlist + `bulk_insert_*` (M2), `build_resolution_map`/`UF` (M3), `retrieve_graph_context` (M4), `judge_answer`/`run_evaluation` (M5).

Bài học chính là kiểm tra dependency theo từng bước nhỏ: token/access, model availability, single-chunk API test, kết nối Aura, rồi mới chạy batch. Benchmark chỉ hợp lệ khi Golden, chunks và graph cùng corpus.

Với giả định đồ án là trợ lý phân tích tin tức công nghệ, GraphRAG phù hợp truy vấn quan hệ đầu tư, M&A, nhân sự và công nghệ đa bước. Node dự kiến gồm Company, Person, Technology, Article, Event; relations gồm FOUNDED, INVESTED_IN, ACQUIRED, DEVELOPED, PARTNERED_WITH, MENTIONED_IN. Hệ thống sẽ dùng aliases + ANN + numeric-token guard, và temporal cap/community summary để xử lý super-node.

## Tự đánh giá

| Tiêu chí | Điểm (1–5) | Ghi chú |
|---|---:|---|
| Hiểu GraphRAG | 4 | Có thể giải thích provenance, traversal và giới hạn data alignment. |
| Kiểm soát AI Coding Agent | 4 | Giữ scale guard, không dùng O(N²), kiểm tra outputs. |
| Chất lượng graph | 3 | Provenance đầy đủ; cần sửa numeric false merge và tăng coverage extraction. |
| Phân tích/debug | 4 | Đã xử lý gated access, model 404, 429 và Aura reset. |
