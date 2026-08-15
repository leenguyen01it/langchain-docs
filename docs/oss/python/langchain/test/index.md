# Kiểm thử (Test)

> Các chiến lược kiểm thử agent LangChain, bao gồm unit test, integration test, và trajectory evaluation.

Ứng dụng agentic cho phép LLM tự quyết định bước tiếp theo để giải quyết vấn đề. Tính linh hoạt này rất mạnh, nhưng bản chất "hộp đen" của model khiến việc dự đoán một thay đổi nhỏ ở một phần của agent sẽ ảnh hưởng thế nào đến toàn bộ hệ thống trở nên khó khăn. Để xây dựng agent sẵn sàng cho production, việc kiểm thử kỹ lưỡng là bắt buộc.

Có một vài cách tiếp cận để kiểm thử agent của bạn:

* **Unit test** kiểm tra các phần nhỏ, deterministic (xác định) của agent một cách độc lập, dùng in-memory fake để assert hành vi chính xác một cách nhanh chóng và ổn định.
* **Integration test** kiểm thử agent bằng các lệnh gọi mạng thật để xác nhận các thành phần hoạt động cùng nhau, credential và schema khớp nhau, và độ trễ (latency) chấp nhận được.
* **Evals** dùng evaluator để đánh giá trajectory (đường đi thực thi) của agent, thông qua so khớp deterministic hoặc LLM judge (LLM đóng vai giám khảo).

Ứng dụng agentic thường thiên về integration test nhiều hơn vì chúng nối chuỗi nhiều thành phần với nhau và phải xử lý tính không ổn định (flakiness) do bản chất nondeterministic của LLM.

!!! tip "Mẹo"
    Chạy evaluation ở quy mô lớn, theo dõi kết quả theo thời gian, và so sánh các experiment với [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-test-index). Xem [Evaluate an LLM application](https://docs.langchain.com/langsmith/evaluate-llm-application) để bắt đầu.

**Unit testing**: [Mock chat model và dùng in-memory persistence để kiểm thử logic agent mà không cần gọi API](unit-testing.md)

**Integration testing**: [Kiểm thử agent với API LLM thật. Tổ chức test, quản lý key, xử lý flakiness, và kiểm soát chi phí](integration-testing.md)

**Evals**: [Đánh giá trajectory của agent bằng so khớp deterministic hoặc evaluator LLM-as-judge](evals.md)
