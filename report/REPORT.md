# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Sinh viên AI20K  
**Nhóm:** Internal Knowledge Assistant  
**Ngày:** 2026-06-05

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**  
Hai đoạn text có high cosine similarity khi vector embedding của chúng trỏ gần cùng hướng, nghĩa là chúng có nhiều tín hiệu ngữ nghĩa hoặc từ khóa chung. Điểm càng gần 1 thì hai đoạn càng giống nhau theo biểu diễn vector.

**Ví dụ HIGH similarity:**
- Sentence A: Python is widely used for data analysis and machine learning.
- Sentence B: Python supports data analysis, automation, and ML workflows.
- Tại sao tương đồng: Cả hai cùng nói về Python và các use case trong data/ML.

**Ví dụ LOW similarity:**
- Sentence A: Vector stores retrieve embeddings by similarity.
- Sentence B: The recipe calls for boiling pasta in salted water.
- Tại sao khác: Hai câu thuộc hai domain khác nhau, gần như không chia sẻ ý nghĩa chính.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**  
Cosine similarity tập trung vào hướng của vector nên phù hợp để so sánh ý nghĩa, trong khi Euclidean distance bị ảnh hưởng mạnh bởi độ lớn vector. Với text embeddings, hướng thường quan trọng hơn magnitude.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**  
Công thức: `ceil((doc_length - overlap) / (chunk_size - overlap))`  
Tính: `ceil((10000 - 50) / (500 - 50)) = ceil(9950 / 450) = 23`  
Đáp án: **23 chunks**

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**  
Tính: `ceil((10000 - 100) / (500 - 100)) = ceil(9900 / 400) = 25`, nên số chunk tăng từ 23 lên **25 chunks**. Overlap nhiều hơn giúp giữ ngữ cảnh giữa hai chunk liền kề, nhưng làm tăng số chunk cần embed và search.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Internal knowledge assistant / RAG support documentation

**Tại sao nhóm chọn domain này?**  
Domain này phù hợp với bài lab vì có đủ tài liệu về Python, vector store, RAG, chunking, support playbook và ghi chú tiếng Việt. Các tài liệu có cấu trúc rõ ràng, dễ gắn metadata theo `category`, `language`, và `source`, nên có thể đánh giá cả semantic search lẫn metadata filtering.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | Python Intro | `data/python_intro.txt` | 1,944 | `category=programming`, `language=en`, `source=data/python_intro.txt` |
| 2 | Vector Store Notes | `data/vector_store_notes.md` | 2,123 | `category=vector_store`, `language=en`, `source=data/vector_store_notes.md` |
| 3 | RAG System Design | `data/rag_system_design.md` | 2,391 | `category=rag`, `language=en`, `source=data/rag_system_design.md` |
| 4 | Customer Support Playbook | `data/customer_support_playbook.txt` | 1,692 | `category=support`, `language=en`, `source=data/customer_support_playbook.txt` |
| 5 | Chunking Experiment Report | `data/chunking_experiment_report.md` | 1,987 | `category=chunking`, `language=en`, `source=data/chunking_experiment_report.md` |
| 6 | Vietnamese Retrieval Notes | `data/vi_retrieval_notes.md` | 1,667 | `category=retrieval`, `language=vi`, `source=data/vi_retrieval_notes.md` |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| `category` | string | `rag`, `support`, `chunking` | Lọc search theo domain câu hỏi để giảm nhiễu. |
| `language` | string | `en`, `vi` | Tránh retrieve nhầm tài liệu khác ngôn ngữ khi user hỏi rõ scope. |
| `source` | string | `data/rag_system_design.md` | Truy vết câu trả lời về tài liệu gốc. |
| `chunk_index` | int | `4` | Debug xem chunk nào được retrieve và có cần chỉnh chunking không. |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` với `chunk_size=500`:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| Python Intro | FixedSizeChunker (`fixed_size`) | 5 | 428.8 | Trung bình, đôi khi cắt giữa câu |
| Python Intro | SentenceChunker (`by_sentences`) | 3 | 645.7 | Tốt nhưng chunk hơi dài |
| Python Intro | RecursiveChunker (`recursive`) | 5 | 387.0 | Tốt, giữ đoạn tự nhiên |
| Vector Store Notes | FixedSizeChunker (`fixed_size`) | 5 | 464.6 | Trung bình |
| Vector Store Notes | SentenceChunker (`by_sentences`) | 5 | 422.4 | Tốt |
| Vector Store Notes | RecursiveChunker (`recursive`) | 7 | 301.4 | Tốt nhất cho section/paragraph |
| RAG System Design | FixedSizeChunker (`fixed_size`) | 6 | 440.2 | Trung bình |
| RAG System Design | SentenceChunker (`by_sentences`) | 3 | 794.0 | Coherent nhưng quá dài |
| RAG System Design | RecursiveChunker (`recursive`) | 7 | 339.7 | Cân bằng nhất |

### Strategy Của Tôi

**Loại:** RecursiveChunker

**Mô tả cách hoạt động:**  
Tôi dùng `RecursiveChunker(chunk_size=500)` để ưu tiên tách theo ranh giới lớn trước: đoạn văn, dòng mới, câu, rồi dấu cách. Nếu một phần vẫn quá dài, thuật toán tiếp tục recurse với separator nhỏ hơn. Khi không còn separator phù hợp, nó fallback sang chia fixed-size theo ký tự.

**Tại sao tôi chọn strategy này cho domain nhóm?**  
Tài liệu nhóm là markdown/text có nhiều section và paragraph, nên recursive chunking giữ được ý hoàn chỉnh tốt hơn fixed-size. So với sentence chunking, recursive chunking kiểm soát độ dài đều hơn, tránh một chunk quá dài chứa nhiều ý khác nhau.

**Code snippet (nếu custom):**
```python
chunker = RecursiveChunker(chunk_size=500)
chunks = chunker.chunk(document_text)
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| Mixed sample docs | best baseline: SentenceChunker | 11 chunks trên 3 docs mẫu | 620.7 | Dễ đọc nhưng một số chunk dài |
| Mixed sample docs | **của tôi: RecursiveChunker** | 19 chunks trên 3 docs mẫu | 342.7 | Cân bằng hơn, top-3 relevant tốt hơn |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Tôi | RecursiveChunker + metadata filter | 10 | Giữ paragraph, filter đúng category | Nhiều chunk hơn fixed-size |
| Thành viên Fixed-size | FixedSizeChunker 500/50 | 8 | Predictable, đơn giản | Có lúc cắt giữa câu |
| Thành viên Sentence | SentenceChunker 2 sentences/chunk | 9 | Chunk rất dễ đọc | Khó kiểm soát độ dài khi câu dài |

**Strategy nào tốt nhất cho domain này? Tại sao?**  
RecursiveChunker là tốt nhất cho bộ tài liệu này vì dữ liệu có cấu trúc paragraph/markdown rõ ràng. Nó giữ context tốt hơn fixed-size và không tạo chunk quá dài như sentence chunking.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk` — approach:**  
Tôi dùng regex `(?<=[.!?])(?:\s+|\n+)` để tách câu sau dấu `.`, `!`, `?` và newline. Sau đó group tối đa `max_sentences_per_chunk` câu vào một chunk, đồng thời strip whitespace và bỏ chunk rỗng.

**`RecursiveChunker.chunk` / `_split` — approach:**  
Base case là text rỗng hoặc text đã ngắn hơn `chunk_size`. Nếu text quá dài, `_split` thử separator theo thứ tự ưu tiên; piece nào vẫn quá dài sẽ recurse với separator nhỏ hơn, còn các piece ngắn được pack lại đến khi gần `chunk_size`.

### EmbeddingStore

**`add_documents` + `search` — approach:**  
Mỗi `Document` được normalize thành record gồm `id`, `content`, `metadata`, `doc_id`, và `embedding`. Search embed query một lần rồi tính dot product với từng record, sort giảm dần theo score và trả về tối đa `top_k`.

**`search_with_filter` + `delete_document` — approach:**  
`search_with_filter` lọc metadata trước, sau đó mới rank để tránh top-k bị chiếm bởi tài liệu sai category. `delete_document` xóa tất cả record có `doc_id` khớp với document cần xóa và trả về `True/False` theo việc có xóa được hay không.

### KnowledgeBaseAgent

**`answer` — approach:**  
Agent retrieve top-k chunk từ store, format mỗi chunk thành block có source và score, rồi inject toàn bộ context vào prompt. Prompt yêu cầu LLM chỉ trả lời dựa trên context và nói rõ nếu context không đủ.

### Test Results

```text
python -m pytest tests/ -v
collected 42 items
42 passed in 0.07s
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | Python supports data analysis and machine learning workflows. | Python is widely used for data analysis and machine learning. | high | 0.750 | Có |
| 2 | Vector stores keep embeddings and metadata for similarity search. | A vector store retrieves embeddings by similarity and stores metadata. | high | 0.408 | Có |
| 3 | Metadata filters narrow retrieval to the right department. | The recipe calls for boiling pasta in salted water. | low | 0.105 | Có |
| 4 | Recursive chunking preserves context by splitting paragraphs first. | Recursive chunking splits on paragraphs before smaller separators. | high | 0.375 | Có |
| 5 | Customer support should escalate when retrieval is insufficient. | Billing documents explain password recovery steps. | low | 0.000 | Có |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**  
Pair 2 và 4 đúng hướng nhưng score không quá cao vì mock embedder là token-hashing đơn giản, không hiểu synonym sâu như model thật. Điều này nhắc rằng embedding backend ảnh hưởng trực tiếp đến retrieval quality; strategy tốt vẫn cần embedding phù hợp.

---

## 6. Results — Cá nhân (10 điểm)

Tôi dùng `RecursiveChunker(chunk_size=500)`, tạo 34 chunks từ 6 documents, lưu vào `EmbeddingStore`, rồi chạy 5 benchmark queries. Các query có scope rõ được chạy bằng `search_with_filter()` theo `category`.

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | What are the four stages in the vector search pipeline? | Chunk documents, embed each chunk, store vectors with metadata, then embed the query and rank by similarity. |
| 2 | How can metadata filters improve retrieval precision? | Metadata filters narrow the search space by source, language, product area, department, or access level, reducing noisy results. |
| 3 | What should a support assistant do when retrieval is insufficient for a new billing issue? | It should recommend escalation instead of improvising a risky answer. |
| 4 | Why is recursive chunking a strong default for mixed technical documentation? | It splits by larger structures first, such as paragraphs, then falls back to smaller separators, preserving context while controlling chunk size. |
| 5 | How should a RAG assistant answer when retrieved context is weak or contradictory? | It should say the evidence is insufficient or contradictory rather than pretending the answer is complete. |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | four stages in vector search pipeline | Vector Store Notes intro; top-3 also included workflow list | 0.420 | Yes, top-3 | Pipeline gồm chunk, embed, store vector/metadata, embed query và rank. |
| 2 | metadata filters improve retrieval precision | Metadata Matters section in Vector Store Notes | 0.250 | Yes | Filter theo source/language/department giúp giảm noise và tăng precision. |
| 3 | support assistant insufficient retrieval billing issue | Support playbook chunk about new billing issue and escalation | 0.605 | Yes | Escalate thay vì tự bịa câu trả lời thiếu căn cứ. |
| 4 | recursive chunking default mixed technical docs | Chunking report; top-3 included recursive default conclusion | 0.401 | Yes, top-3 | Recursive chunking giữ paragraph trước rồi tách nhỏ khi cần. |
| 5 | RAG assistant weak or contradictory context | RAG design chunk about grounding and insufficient evidence | 0.405 | Yes | Nếu context yếu/mâu thuẫn, assistant phải nói rõ giới hạn. |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**  
Fixed-size chunking rất dễ debug vì chunk count predict được, nhưng không nên dùng một cách máy móc cho tài liệu có cấu trúc rõ. Sentence chunking giúp chunk dễ đọc, tuy nhiên cần kiểm tra độ dài khi câu hoặc paragraph quá dài.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**  
Một nhóm khác dùng metadata `audience=public/internal` cho support docs, và filter này giúp tránh lộ nội dung nội bộ trong câu trả lời customer-facing. Đây là ví dụ rõ metadata không chỉ tăng precision mà còn giúp kiểm soát rủi ro.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**  
Tôi sẽ thêm nhiều tài liệu có cùng chủ đề nhưng khác audience/date để test metadata filtering khó hơn. Tôi cũng sẽ tạo benchmark có một failure case cố ý, ví dụ query mơ hồ hoặc tài liệu cũ cạnh tranh với tài liệu mới.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 10 / 10 |
| Chunking strategy | Nhóm | 15 / 15 |
| My approach | Cá nhân | 10 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 10 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 5 / 5 |
| **Tổng** | | **100 / 100** |
