# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Nguyễn Hữu Huy  
**Vai trò trong nhóm:** Documentation Owner  
**Ngày nộp:** 13/04/2026  


---

## 1. Tôi đã làm gì trong lab này? (150 từ)

Trong lab này, với vai trò Documentation Owner, tôi chịu trách nhiệm chính về việc thiết kế cấu trúc dữ liệu (indexing) và ghi chép toàn bộ quá trình phát triển hệ thống. Trong Sprint 1, tôi đã phối hợp với Tech Lead để đưa ra quyết định về chiến lược chunking (400 tokens, 80 overlap) và định nghĩa bộ Metadata gồm 5 trường (`source`, `section`, `effective_date`, `department`, `access`) để phục vụ cho việc lọc và trích dẫn nguồn. 

Trong Sprint 3 và 4, tôi tập trung vào việc hoàn thiện `docs/architecture.md` và `docs/tuning-log.md`. Tôi đã trực tiếp thực hiện việc so sánh kết quả giữa hai cấu hình Baseline (Dense) và Variant (Hybrid), phân tích các metric từ scorecard để rút ra kết luận về hiệu quả của từng phương pháp. Ngoài ra, tôi hỗ trợ Eval Owner trong việc chạy script chấm điểm tự động và tổng hợp `logs/grading_run.json` để đảm bảo báo cáo cuối cùng phản ánh chính xác khả năng thực tế của pipeline.

---

## 2. Điều tôi hiểu rõ hơn sau lab này (150 từ)

Sau lab này, tôi đã hiểu sâu sắc hơn về tầm quan trọng của **Metadata-driven Retrieval** và sự đánh đổi giữa **Dense vs Hybrid Retrieval**. Trước đây, tôi nghĩ rằng cứ dùng Embedding Model mạnh (như OpenAI) là đủ. Tuy nhiên, khi thực hiện Lab, tôi nhận ra rằng Metadata không chỉ để dẫn nguồn mà còn là "la bàn" để LLM định vị đúng phiên bản tài liệu (như chính sách v3 vs v4). 

Đặc biệt, qua việc phân tích `tuning-log.md`, tôi hiểu rằng **Hybrid Retrieval** không phải lúc nào cũng là "viên đạn bạc". Trong tập dữ liệu nhỏ (~30 chunks), Dense search của OpenAI đã quá xuất sắc trong việc nắm bắt ngữ nghĩa, trong khi Hybrid đôi khi làm nhiễu kết quả do trọng số của keyword search quá cao, dẫn đến điểm Completeness bị giảm ở một số câu hỏi phức tạp. Điều này dạy cho tôi bài học về việc phải luôn "data-driven" khi tối ưu RAG thay vì áp dụng các kỹ thuật phức tạp một cách máy móc.

---

## 3. Điều ngạc nhiên hoặc gặp khó khăn (150 từ)

Điều làm tôi ngạc nhiên nhất là khả năng **abstain (từ chối trả lời)** của model khi gặp thông tin thiếu. Ban đầu, tôi lo lắng model sẽ "bịa" (hallucinate) ra các quy định không có thật để làm hài lòng người dùng. Tuy nhiên, nhờ việc tinh chỉnh prompt "Answer only from context" và thiết lập `temperature = 0`, hệ thống đã trả lời "Tôi không biết" rất dứt khoát ở câu `gq07` (về mức phạt SLA) và `q10` (về hoàn tiền VIP). 

Khó khăn lớn nhất  gặp phải là việc debug lỗi **OpenAI API 400** ở câu `gq03`. Sau khi kiểm tra, phát hiện ra vấn đề nằm ở việc xử lý các ký tự đặc biệt trong context khi build prompt, khiến JSON gửi lên bị lỗi. Lỗi này mất hơn 20 phút để trace vì ban đầu tập trung vào logic retrieval mà quên mất bước validation cuối cùng trước khi gọi LLM. Đây là kinh nghiệm quý báu về việc xử lý ngoại lệ trong pipeline sản xuất.

---

## 4. Phân tích một câu hỏi trong grading (200 từ)

Phân tích câu **gq06**: *"Lúc 2 giờ sáng xảy ra sự cố P1, on-call engineer cần cấp quyền tạm thời cho một engineer xử lý incident. Quy trình cụ thể như thế nào và quyền này tồn tại bao lâu?"*

**Phân tích:**
- **Kết quả:** Pipeline trả lời rất xuất sắc (Faithfulness 5/5, Completeness 5/5).
- **Quy trình xử lý:** Đây là một câu hỏi "multi-hop" đòi hỏi thông tin từ hai nguồn khác nhau: `access_control_sop.txt` (quy trình cấp quyền tạm thời) và `sla_p1_2026.txt` (định nghĩa sự cố P1 và hotline on-call). 
- **Tại sao thành công:** Nhờ chiến lược Hybrid retrieval trong Variant, hệ thống đã bắt được các từ khóa cực kỳ đặc hiệu như "2 giờ sáng", "P1", "on-call". 
- **Chi tiết kỹ thuật:** Model đã trích xuất đúng điều khoản "phê duyệt bằng lời từ Tech Lead" và "giới hạn 24 giờ" — những chi tiết rất dễ bị bỏ sót nếu chunking size quá nhỏ hoặc không có overlap. Đặc biệt, việc metadata `section` được gắn đúng đã giúp LLM định vị được đây là quy trình xử lý sự cố khẩn cấp chứ không phải quy trình cấp quyền Admin thông thường (Level 4). Đây là minh chứng cho thấy sự kết hợp giữa chunking thông minh và retrieval đa phương thức giúp giải quyết các case phức tạp rất tốt.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì? (100 từ)

Nếu có thêm thời gian, tôi sẽ đề xuất thực hiện **Reranking với Cross-Encoder** (như `BGE-Reranker`). Dựa trên bằng chứng từ Scorecard, điểm Completeness của chúng ta vẫn còn dao động quanh mức 3.0 - 3.6/5. Điều này cho thấy Top-3 chunks hiện tại đôi khi chứa các chunk "gần đúng" nhưng không chứa đủ toàn bộ chi tiết cần thiết. Việc tăng `top_k_search` lên 20 và dùng Reranker để chọn ra 3 chunk "đậm đặc" nhất chắc chắn sẽ giúp cải thiện điểm Completeness mà không làm loãng prompt. Ngoài ra, tôi muốn thử nghiệm **Query Expansion** để xử lý các câu hỏi dùng từ lóng hoặc từ viết tắt mà trong docs chưa cập nhật hết.

---

