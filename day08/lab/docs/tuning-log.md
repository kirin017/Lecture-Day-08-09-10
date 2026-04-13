# Tuning Log — RAG Pipeline (Day 08 Lab)

> Template: Ghi lại mỗi thay đổi và kết quả quan sát được.
> A/B Rule: Chỉ đổi MỘT biến mỗi lần.

---

## Baseline (Sprint 2)

**Ngày:** 2026-04-13  
**Config:**
```
retrieval_mode = "dense"
chunk_size = 400 tokens
overlap = 80 tokens
top_k_search = 10
top_k_select = 3
use_rerank = False
llm_model = "gpt-4o-mini"
```

**Scorecard Baseline:**
| Metric | Average Score |
|--------|--------------|
| Faithfulness | 4.90 /5 |
| Answer Relevance | 4.80 /5 |
| Context Recall | 5.00 /5 |
| Completeness | 3.60 /5 |

**Câu hỏi yếu nhất (điểm thấp):**
1. **q07 (Approval Matrix)** - Completeness = 2/5. Mặc dù retrieval đúng context nhưng LLM chưa trích xuất được thông tin về việc tài liệu đã đổi tên một cách rõ ràng như mong đợi.
2. **q10 (Hoàn tiền VIP)** - Completeness = 3/5. Model nhận diện đúng là không có thông tin cụ thể nhưng phần giải thích chưa bao quát hết các khía cạnh của chính sách hiện hành.

**Giả thuyết nguyên nhân (Error Tree):**
- [ ] Indexing: Chunking cắt giữa điều khoản
- [x] Retrieval: Dense search đã làm rất tốt (Recall 5.0)
- [x] Generation: Completeness thấp do prompt chưa khuyến khích model phân tích sâu các trường hợp ngoại lệ hoặc sự thay đổi thông tin (alias).

---

## Variant 1 (Sprint 3)

**Ngày:** 2026-04-13  
**Biến thay đổi:** `retrieval_mode = "hybrid"`  
**Lý do chọn biến này:**
Chọn hybrid vì q07 liên quan đến việc tìm kiếm theo alias/tên cũ ("Approval Matrix"). Giả thuyết là hybrid (kết hợp keyword search) sẽ giúp mang lại context có độ liên quan cao hơn cho các câu hỏi chứa keyword đặc thù.

**Config thay đổi:**
```
retrieval_mode = "hybrid"
# Các tham số còn lại giữ nguyên như baseline
```

**Scorecard Variant 1:**
| Metric | Baseline | Variant 1 | Delta |
|--------|----------|-----------|-------|
| Faithfulness | 4.90/5 | 5.00/5 | +0.10 |
| Answer Relevance | 4.80/5 | 4.70/5 | -0.10 |
| Context Recall | 5.00/5 | 5.00/5 | 0.00 |
| Completeness | 3.60/5 | 3.00/5 | -0.60 |

**Nhận xét:**
- Variant 1 (Hybrid) cải thiện nhẹ về tính trung thực (Faithfulness) nhờ việc truy xuất context có keyword chính xác hơn, giúp LLM ít phải suy luận ngoài context.
- Tuy nhiên, Completeness bị giảm đáng kể ở một số câu (như q06 và q09). Đặc biệt ở q09 (ERR-403), variant hybrid khiến model trả về "Tôi không biết" (abstain hoàn toàn), dẫn đến điểm completeness thấp nhưng faithfulness tuyệt đối.

**Kết luận:**
Variant 1 không thực sự tốt hơn baseline về mặt tổng thể (điểm completeness giảm). Trong corpus quy mô nhỏ này, dense search của OpenAI (`text-embedding-3-small`) đã đủ mạnh để xử lý hầu hết các query kể cả query có alias. Hybrid có thể hữu ích hơn khi corpus lớn hơn hoặc chứa nhiều mã kỹ thuật cực kỳ đặc thù mà embedding model chưa được fine-tune.

---

## Variant 2 (nếu có thời gian)

**Biến thay đổi:** Rerank (Cross-Encoder)
**Config:**
```
use_rerank = True
top_k_search = 20
top_k_select = 3
```

---

## Tóm tắt học được

1. **Lỗi phổ biến nhất trong pipeline này là gì?**
   > Lỗi Completeness: Model thường trả lời đúng trọng tâm nhưng bỏ lỡ các chi tiết phụ hoặc không giải thích rõ ràng các trường hợp "không tìm thấy thông tin" (abstain).

2. **Biến nào có tác động lớn nhất tới chất lượng?**
   > Trong bài lab này, chất lượng của Embedding Model (dense retrieval) và Prompt Engineering có tác động lớn nhất. Việc đổi sang Hybrid không mang lại cải thiện đột phá do tập dữ liệu nhỏ.

3. **Nếu có thêm 1 giờ, nhóm sẽ thử gì tiếp theo?**
   > Thử nghiệm Reranking (Cross-Encoder) sau khi retrieve để lọc ra 3 chunk thực sự chất lượng nhất, và tinh chỉnh Grounded Prompt để cải thiện khả năng trả lời các câu hỏi về alias hoặc thiếu context.
