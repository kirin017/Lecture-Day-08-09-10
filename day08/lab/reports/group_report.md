# Group Report — Lab Day 08: RAG Pipeline

---

## 1. Mục tiêu và Phạm vi dự án

Dự án tập trung xây dựng một hệ thống RAG (Retrieval-Augmented Generation) phục vụ khối CS và IT Helpdesk. Mục tiêu chính là cung cấp câu trả lời chính xác, có dẫn nguồn (citations) từ 5 tài liệu chính sách nội bộ (Refund, SLA, Access Control, IT FAQ, HR Leave). Hệ thống ưu tiên tính **Trung thực (Faithfulness)** và khả năng **Từ chối (Abstain)** khi không có đủ dữ liệu để tránh lỗi nghiêm trọng nhất là ảo giác (hallucination).

---

## 2. Quyết định kỹ thuật trọng tâm

### 2.1. Chiến lược Indexing & Metadata
Nhóm thống nhất sử dụng chiến lược **Section-based Chunking** kết hợp giới hạn kích thước (400 tokens, 80 tokens overlap). 
- **Lý do:** Đảm bảo các điều khoản pháp lý hoặc quy trình kỹ thuật không bị cắt đôi một cách vô nghĩa, đồng thời duy trì đủ ngữ cảnh cho LLM ở các đoạn chuyển tiếp.
- **Metadata:** Mỗi chunk được làm giàu với 5 trường dữ liệu: `source`, `section`, `effective_date`, `department`, và `access`. Điều này cho phép hệ thống thực hiện trích dẫn chính xác và hỗ trợ lọc theo tính thời điểm (freshness) của chính sách.

### 2.2. Chiến lược Retrieval: Dense vs. Hybrid
Nhóm đã thử nghiệm hai phương thức:
- **Baseline (Dense):** Sử dụng `text-embedding-3-small` của OpenAI.
- **Variant (Hybrid):** Kết hợp Dense search và BM25 (keyword search).

**Kết quả quan sát:** Mặc dù Hybrid retrieval giúp bắt được các từ khóa kỹ thuật (như mã lỗi `ERR-403`), nhưng trong quy mô corpus nhỏ, nó lại làm giảm điểm **Completeness (Tính đầy đủ)**. Nguyên nhân được xác định là do keyword search đôi khi đẩy các chunk chứa từ khóa trùng lặp lên trên, nhưng các chunk này lại thiếu ngữ cảnh bổ trợ mà Dense search có thể bao quát được.

### 2.3. Prompt Engineering & Grounding
Chúng tôi áp dụng mô hình **Grounded Prompt** với các ràng buộc nghiêm ngặt:
- Chỉ trả lời dựa trên context được cung cấp.
- Trả về "Tôi không biết" nếu context thiếu thông tin.
- Định dạng dẫn nguồn chuẩn `[1]`.
- Thiết lập `temperature = 0` để đảm bảo kết quả nhất quán trong quá trình đánh giá (Evaluation).

---

## 3. Phân tích kết quả thực tế (Grading Questions)

Dựa trên kết quả chạy `logs/grading_run.json`, hệ thống đạt được các chỉ số sau:

| Chỉ số (Trung bình) | Giá trị | Nhận xét |
|---------------------|---------|----------|
| **Faithfulness** | 4.9 / 5 | Rất cao, hệ thống bám sát context, không bịa đặt. |
| **Context Recall** | 5.0 / 5 | Retrieval hoạt động hoàn hảo, tìm đúng tài liệu mục tiêu. |
| **Completeness** | 3.2 / 5 | Mức trung bình. Hệ thống trả lời đúng ý chính nhưng bỏ sót chi tiết phụ. |

**Điểm sáng:**
- **Abstain thành công:** Ở câu `gq07` (về mức phạt SLA), hệ thống từ chối trả lời một cách chính xác vì tài liệu không quy định. Điều này giúp nhóm tránh được lỗi Penalty (hallucination).
- **Multi-document Reasoning:** Ở câu `gq06`, hệ thống đã tổng hợp thành công thông tin từ hai tài liệu khác nhau để đưa ra quy trình xử lý sự cố P1 hoàn chỉnh.

**Hạn chế (Failure Modes):**
- **Lỗi kỹ thuật:** Câu `gq03` bị lỗi `OpenAI API 400` do vấn đề parse JSON khi context chứa các ký tự đặc biệt không được xử lý tốt.
- **Tính đầy đủ:** Một số câu hỏi phức tạp về mặt con số (như `gq05` về số ngày phê duyệt Admin access) bị model rút gọn quá mức, dẫn đến mất các điều kiện cần thiết.

---

## 4. Bài học kinh nghiệm & Đề xuất cải tiến

1. **Chất lượng dữ liệu đầu vào:** Việc tiền xử lý tài liệu (strip các header/footer thừa) và chunking theo cấu trúc tài liệu (Semantic Chunking) quan trọng hơn nhiều so với việc tinh chỉnh các thuật toán retrieval phức tạp.
2. **Ưu tiên Reranking:** Thay vì chỉ dựa vào Hybrid search, nhóm nhận thấy việc sử dụng một mô hình **Reranker (Cross-Encoder)** sau bước retrieval sẽ giúp lọc ra các chunk thực sự chất lượng nhất cho prompt, từ đó cải thiện đáng kể điểm Completeness.
3. **Cải tiến Prompt:** Cần tinh chỉnh prompt để khuyến khích model trích xuất "toàn bộ các điều kiện và ngoại lệ" thay vì chỉ đưa ra câu trả lời ngắn gọn nhất.

---

## 5. Kết luận

Hệ thống RAG hiện tại đã đạt được mục tiêu cơ bản: **Chính xác, An toàn, và Có trích dẫn**. Mặc dù còn hạn chế về tính đầy đủ trong một số trường hợp phức tạp, nhưng nền tảng về Indexing và Retrieval hiện tại là đủ vững chắc để mở rộng cho các bài toán quy mô lớn hơn của doanh nghiệp.
