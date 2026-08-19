# Thuyết minh kỹ thuật — Lab 19 GraphRAG vs Flat RAG

## 1. Coreference Resolution

Một chunk có `unresolved_mentions = ["we", "our users'", "These"]`. Các đại từ này không có antecedent duy nhất trong cùng chunk, nên pipeline giữ nguyên thay vì thay thế. Đây là lựa chọn conservative: thay nhầm "we" cho một công ty sẽ tạo false edge và làm traversal trả về quan hệ không có bằng chứng.

## 2. Ngưỡng Entity Resolution

Ngưỡng merge vector là cosine `0.90`; sau đó bắt buộc qua lexical guard với SequenceMatcher `>= 0.72` sau khi bỏ hậu tố công ty. Ngưỡng cao ưu tiên precision vì false merge nguy hiểm hơn bỏ sót một alias.

## 3. False merge quan sát được

`Sonos Era 100` và `Sonos Era 300` có similarity `0.903168` và bị `MERGE_VECTOR`. Đây là false merge: hai số model khác nhau chỉ hai sản phẩm khác nhau. Guard hiện tại không kiểm tra numeric token. Cải tiến cần thiết là reject khi tập số trong hai tên khác nhau, trước lexical similarity.

## 4. Audit Entity Resolution

Audit cuối có 130 dòng: 1 `MERGE_VECTOR` và 129 `REJECT_THRESHOLD`. Ví dụ `3M` và `3M Co` đạt 0.820720 nhưng bị giữ riêng vì dưới 0.90. Audit lưu `type`, `left`, `right`, `similarity`, `decision`, `reason`, tạo khả năng truy vết các quyết định hợp nhất/từ chối.

## 5. Bulk ingestion

Nodes và edges được đưa vào Neo4j bằng `UNWIND $rows AS row`; không insert từng row. Trong lần chạy Aura có một lần socket bị ngắt, sau đó chạy lại với batch size 100. Do `MERGE` idempotent, thao tác chạy lại không tạo bản sao.

## 6. Provenance Integrity

Mỗi edge giữ `source_chunk_id`, `published_date`, `evidence`, `confidence`. Sanity check cuối trả về 89 nodes, 55 edges và `invalid_provenance_edges = 0`. Đây là điều kiện để câu trả lời GraphRAG có thể dẫn chứng về chunk nguồn.

## 7. Super-node Mitigation

Top 3 degree là 1Kosmos (4), OnePlus (3), Worldcoin (3); sample chưa có node degree >100. Chính sách vẫn được cài: node vượt 100 chỉ lấy tối đa 50 edge mới nhất, toàn traversal cap 250 edge và graph context cap 14,000 ký tự. Ưu điểm là kiểm soát latency/token; rủi ro là mất evidence lịch sử cũ.

## 8. Flat RAG và Hybrid GraphRAG

FAISS `IndexFlatIP` index 1,500 chunks. Graph retrieval trích seed, exact/fuzzy match entity, BFS tối đa 2 hop rồi textualize edge với provenance; sau đó ghép graph context với 4 vector chunks. Test `Which company partnered with Transact?` match seed Transact, mở rộng 2 nodes, lấy 1 edge và trả lời Talkiatry cho cả hai kiến trúc.

## 9. Benchmark và failure mode

Evaluation cuối dùng 5 câu trong `data/golden_dataset.csv`: 1 factoid, 2 multi-hop, 2 cross-doc; mọi evidence chunk đều thuộc index/graph. Multi-hop đạt 5/5 cho cả hai kiến trúc. Hai câu cross-doc về 1Kosmos có GraphRAG 5/5 trên cả ba metric, còn Flat RAG 1/5 vì không thu được hai chunk tháng 8 và tháng 10 cùng lúc. Đổi lại, GraphRAG cross-doc chậm hơn 6.031s so với 5.201s và tốn 957.5 so với 849.5 token. Ca factoid Transact cho kết quả ngược lại: Flat 5/5, Graph 1/5 dù edge tồn tại, cho thấy cần regression test seed/traversal và vector fallback. Bộ Golden `first5000` trước đó là failure data-alignment lịch sử, không dùng để kết luận cuối.

## 10. Chi phí, agent control và scale

Coreference và NER/RE gặp model deprecation (`llama-3.3-70b-versatile`) rồi rate limit 429 với model thay thế. Pipeline đổi sang model Groq khả dụng, nhưng NER chỉ ingest phần triples hoàn tất trước rate limit. Đề xuất pairwise cosine O(N²) đã bị từ chối; hệ thống dùng FAISS ANN + Union-Find. Với 350MB/100k bài, cần queue bất đồng bộ, retry/backoff có checkpoint, HNSW/partitioned ANN, batch UNWIND và community partitioning.
