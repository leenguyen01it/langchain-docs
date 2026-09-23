# Track: Evaluation

Evaluation (eval) là kỹ năng **quan trọng nhất và bị xem nhẹ nhất** của AI Engineer. Không có eval, mọi quyết định (đổi prompt, đổi model, thêm kỹ thuật) đều dựa trên cảm giác từ vài câu thử. Có eval, bạn cải tiến nhanh, tự tin, và chứng minh được giá trị của sản phẩm bằng số liệu.

| Bài | Nội dung | Thời lượng |
|---|---|---|
| [Bài 1: Tư duy eval và phân tích lỗi](01-tu-duy-eval.md) | Vì sao eval khác test, quy trình phân tích lỗi từ dữ liệu thật, xây dataset đầu tiên | 4 đến 5 ngày |
| [Bài 2: Chấm điểm và LLM-as-a-judge](02-llm-judge.md) | Kiểm tra bằng code, giám khảo LLM, thiết kế rubric, kiểm định giám khảo, các thiên lệch | 4 đến 5 ngày |
| [Bài 3: Đánh giá agent](03-eval-agent.md) | Đánh giá kết quả cuối, quỹ đạo, từng bước; người dùng giả lập; LangSmith | 4 đến 5 ngày |
| [Bài 4: Eval online, A/B test và red teaming](04-online-eval.md) | Giám sát chất lượng trên production, feedback, A/B test, kiểm thử tấn công | 4 đến 5 ngày |

**Yêu cầu trước:** [Hiểu LLM](../hieu-llm/index.md). Nếu đã học [RAG Bài 6](../rag/06-evaluation.md), bạn đã quen với metric retrieval và eval runner; track này tổng quát hóa cho mọi loại ứng dụng LLM.

!!! quote "Nguyên tắc số một"
    **Nhìn vào dữ liệu.** Đọc output thật của hệ thống, hàng chục, hàng trăm cái. Mọi metric, mọi giám khảo tự động đều bắt đầu từ việc con người hiểu hệ thống đang sai ở đâu.

---

[Lộ trình AI Engineer](../index.md)
