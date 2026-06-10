# AI & RAG Implementation Strategy

**Project:** SAP IEE2E Learning & RAG Evaluation Platform  
**Target:** Cloudflare Workers + Gemini 2.5 Flash API

---

## 1. Data Pipeline (Document Processing)

To guarantee minimum latency and prevent Cloudflare Worker timeouts, the system strictly processes raw text formats.

### 1.1. Supported Formats & Parsing

- **Formats:** `.md` (Markdown) and `.txt` ONLY. (Support for `.docx` is explicitly excluded to bypass heavy external parsing libraries).
- **Ingestion:** Cloudflare Workers read the file buffer as a raw UTF-8 string.

### 1.2. Semantic Chunking Strategy

Given the highly structured nature of SAP documentation (e.g., `# Unit 1`, `## 1.1 Introduction`, `### 1.2.1 Finance`), blind character-count chunking will destroy context.

- **Methodology:** Markdown-Aware Semantic Chunking. The parser splits text primarily at Heading tags (`#`, `##`, `###`) to ensure that a specific business process or department definition remains intact within a single chunk.
- **Size Limits:** Maximum **1,000 tokens (~700 words)** per chunk.
- **Overlap:** **15% (~150 tokens)** overlap between sequential chunks to preserve narrative continuity.
- **Embedding Model:** `text-embedding-004`.

### 1.3. Retrieval Strategy (Top-K)

- When a Short Answer Question (SAQ) is submitted, the system queries Cloudflare Vectorize using the question's `unit_id` and `topic_id` as metadata filters.
- It retrieves the **Top 3 most relevant chunks** to form the Context Window (max ~3,000 tokens), which is injected into the prompt.

---

## 2. Prompt Engineering Templates

The AI acts as a deterministic, neutral evaluator. It is strictly forbidden from hallucinating or using external knowledge. The evaluation language is strictly Vietnamese.

### 2.1. System Prompt

This prompt configures Gemini's persona and constraints.

```text
Bạn là hệ thống chấm điểm tự động cho kỳ thi SAP IEE2E. Nhiệm vụ của bạn là đánh giá câu trả lời của học viên dựa trên Câu Trả Lời Mẫu của giảng viên và Tài Liệu Tham Khảo (Context) được cung cấp.

NGUYÊN TẮC HOẠT ĐỘNG
1. KHÔNG sử dụng kiến thức bên ngoài hệ thống hoặc tự suy diễn. Chỉ dựa vào Context và Câu Trả Lời Mẫu.
2. Đánh giá nhị phân: Đúng (true) hoặc Sai (false).
3. Văn phong: Trung tính, trực diện, không cảm xúc, không dùng những câu giao tiếp dư thừa (VD: "Tôi hiểu", "Bạn đã làm tốt").
4. Cấu trúc Feedback BẮT BUỘC (Tối đa 2 câu):
   - <Đúng/Sai>.
   - <Chỉ ra chi tiết lỗi sai nếu có, và giải thích ngắn gọn đáp án đúng>.
   - <Ghi chú phần/topic cần xem lại>.
5. Nếu Context cung cấp hoàn toàn không chứa thông tin để kiểm chứng câu trả lời, trả về cờ isConflict = true.
```

### 2.2. User Prompt (Execution Template)

This is dynamically generated per question by the Cloudflare Worker.

```text
ĐÁNH GIÁ CÂU TRẢ LỜI SAU:

[CONTEXT TỪ TÀI LIỆU]
{retrieved_chunks_from_vector_db}

[THÔNG TIN CÂU HỎI]
- Mã câu hỏi: {question_id}
- Nội dung câu hỏi: {question_content}
- Tag phân loại: Unit {unit_tag} / Topic {topic_tag}

[ĐÁP ÁN]
- Câu Trả Lời Mẫu (Tiêu chuẩn): {sample_answer}
- Câu Trả Lời Của Học Viên: {learner_answer}

Dựa vào cấu trúc đã được yêu cầu, hãy chấm điểm và trả kết quả bằng định dạng JSON.
```

---

## 3. Structured Output Schema

To eliminate parsing errors on the backend, the Gemini API is forced into JSON mode using the explicit schema below.

```json
{
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Mã câu hỏi (question_id)"
    },
    "isCorrect": {
      "type": "boolean",
      "description": "Kết quả chấm điểm: true nếu đúng, false nếu sai."
    },
    "feedback": {
      "type": "string",
      "description": "Giải thích chi tiết lỗi sai. Tối đa 2 câu. Format: <Đúng/Sai>. <Lý do>. <Cần xem lại topic nào>."
    },
    "isConflict": {
      "type": "boolean",
      "description": "Đánh dấu true nếu Context không chứa đủ thông tin để chấm điểm dựa trên Câu trả lời mẫu."
    },
    "whyConflict": {
      "type": "string",
      "description": "Nếu isConflict là true, giải thích lý do (VD: 'Context retrieved does not contain enough information to evaluate.'). Nếu false, để chuỗi rỗng."
    }
  },
  "required": ["id", "isCorrect", "feedback", "isConflict", "whyConflict"]
}
```

---

## 4. Fallback & Conflict Handling (Edge Cases)

- Scenario A: Blank Submission
  - If the user submits an empty string for an SAQ, the Worker skips the Gemini API call entirely to save quota and directly writes { "isCorrect": false, "feedback": "Sai. Không có câu trả lời." } to the database.

- Scenario B: RAG Context Miss (The Conflict Flag)
  - If the retrieval system pulls 3 chunks that are irrelevant to the SAQ (e.g., due to missing documentation updates or incorrect topic tags), the AI will detect the discrepancy between the user's valid answer and the void context.

  - Action: AI sets "isConflict": true and writes the reason in "whyConflict". This ensures the user isn't unfairly penalized and flags the specific question ID in the D1 database for Operator review.

- Scenario C: API Rate Limit (429 Error)
  - Handled at the Worker queue layer via Exponential Backoff. The AI is entirely unaware of this. The prompt logic remains deterministic.
