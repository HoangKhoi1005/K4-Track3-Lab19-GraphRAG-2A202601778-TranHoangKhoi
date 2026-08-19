# Reflection & Action Plan — Trần Hoàng Khôi

## Mapping bài giảng vào code

| Khái niệm | Module | Hàm/khối code | Quan sát |
|---|---|---|---|
| Conservative coreference | M1 | `resolve_coref_batch`, `run_coref` | Giữ unresolved mention khi không rõ antecedent. |
| Schema allowlist | M2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Chỉ ingest relation/type cho phép. |
| Bulk ingestion | M2 | `bulk_insert_nodes`, `bulk_insert_edges` | `UNWIND` và `MERGE` giúp retry an toàn. |
| Entity resolution | M3 | `build_resolution_map`, `UF`, FAISS | Có audit; phát hiện false merge Sonos. |
| Super-node control | M4 | `retrieve_graph_context` | Degree cap 100/50, global cap 250. |
| LLM-as-a-Judge | M5 | `judge_answer`, `run_evaluation` | Đánh giá quality cùng latency/token. |

## Debugging và bài học

Lỗi khó nhất là chuỗi phụ thuộc giữa secrets, Hugging Face gated access, model Groq deprecation, rate limit và Neo4j connection reset. Cách xử lý hiệu quả là kiểm tra từng dependency bằng test nhỏ trước khi chạy batch lớn: xác thực token, test một coreference chunk, giảm Neo4j batch size, và dùng checkpoint cho evaluation. Bài học là cần version model và kiểm tra corpus/Golden alignment trước benchmark.

## Action Plan áp dụng thực tế

**Giả định dự án:** trợ lý phân tích tin tức công nghệ cho người dùng cần truy vết quan hệ công ty, đầu tư, công nghệ và nhân sự.

GraphRAG phù hợp cho câu hỏi đa bước như "công ty nào đầu tư vào startup được sáng lập bởi cựu nhân viên X"; Flat RAG vẫn phù hợp tra cứu factoid hoặc tóm tắt đoạn đơn.

- **Nodes:** `Company`, `Person`, `Technology`, `Article`, `Event`.
- **Relations:** `FOUNDED`, `INVESTED_IN`, `ACQUIRED`, `DEVELOPED`, `PARTNERED_WITH`, `MENTIONED_IN`.
- **Entity resolution:** aliases có kiểm duyệt, ANN candidate retrieval, numeric-token/product guard, audit định kỳ.
- **Super-node:** cap theo thời gian, score edge theo recency/relevance, partition/community summaries cho truy vấn global.
