# Failure analysis — Lab 19 GraphRAG vs Flat RAG

## Ca 1: Golden Dataset không cùng corpus index

**Triệu chứng.** Cả Flat RAG và GraphRAG nhận điểm 1 cho hầu hết 5 câu. Ví dụ câu HPE/Axis Security, Amazon/Cohere và Google Cloud Next '23 đều trả lời rằng context không có evidence.

**Root cause.** Golden file `graphrag_golden_50_first5000.csv` được tạo từ tập first5000, còn pipeline lấy random 1,500 bài sau dedup từ file download 300MB. Các chunk được Golden reference trích dẫn không có mặt trong FAISS index và cũng không tạo edge trong Neo4j.

**Tác động.** Graph traversal không thể tạo multi-hop path khi các edge gốc chưa tồn tại. Điểm benchmark không thể dùng để kết luận GraphRAG kém hơn về reasoning.

**Khắc phục.** Đã điền đúng file chuẩn `data/golden_dataset.csv` từ các edge thực trong Neo4j. Bộ mới có 1 factoid, 2 multi-hop và 2 cross-doc; 5/5 evidence `chunk_id` đã được đối chiếu có mặt trong sample 1,500 chunks. Notebook fail-fast nếu chunk ID trong Golden không thuộc `flat_store`. Evaluation aligned đã hoàn tất: hai câu cross-doc có GraphRAG 5/5 trên cả ba metric, trong khi Flat RAG là 1/5 vì chỉ lấy được một phần context. Điều này xác nhận nguyên nhân gốc là corpus mismatch, không phải kết luận rằng GraphRAG không suy luận được.

## Ca 4: GraphRAG bỏ sót factoid Transact

**Triệu chứng.** Ở `GAL-01`, Flat RAG trả lời đúng Talkiatry và đạt 5/5; GraphRAG trả lời rằng context không có partnership với Transact và đạt 1/5. Trong khi đó Neo4j có edge `Talkiatry -PARTNERED_WITH-> Transact` với `source_chunk_id=55a09cbc43c41ffb2dd9::c0000`.

**Root cause.** Đây là rủi ro ở bước entity seed/matching hoặc traversal context, không phải lỗi extraction: GraphRAG không đưa edge phù hợp vào prompt generation dù graph đã lưu provenance.

**Khắc phục.** Thêm regression test cho truy vấn Transact: assert seed match chứa Transact, collected_edges >= 1 và context chứa chunk ID trên trước khi gọi LLM. Khi retrieval graph rỗng, fallback sang top-k vector chunks thay vì kết luận thiếu evidence.

## Ca 2: False merge sản phẩm Sonos

**Triệu chứng.** Audit ghi `Sonos Era 100` và `Sonos Era 300` với similarity 0.903168, decision `MERGE_VECTOR`.

**Root cause.** Lexical guard chỉ dùng string ratio sau khi bỏ suffix công ty; nó không phân biệt model number. Hai chuỗi có phần tiền tố gần giống nên qua guard dù ngữ nghĩa là hai SKU khác nhau.

**Tác động.** Edges của hai sản phẩm bị dồn vào cùng canonical node, làm GraphRAG có thể trả evidence về sai model.

**Khắc phục.** Bổ sung guard numeric-token: nếu hai entity có numeric token khác nhau thì `REJECT_GUARD`; đồng thời không merge các product/technology có hậu tố model khác nhau.

## Ca 3: API deprecation và rate limit

`llama-3.3-70b-versatile` trả 404 do bị deprecate. Sau khi đổi model, Coreference chạy đủ 400 chunks; NER/RE gặp 429 sau một phần batches. Cần version-pin model ID, kiểm tra models trước chạy dài, checkpoint triples/errors, pacing theo RPM/TPM và retry có jitter.
