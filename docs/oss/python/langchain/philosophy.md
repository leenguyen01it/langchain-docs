# Triết lý thiết kế

> LangChain tồn tại để trở thành nơi dễ bắt đầu nhất khi xây dựng với LLM, đồng thời vẫn linh hoạt và sẵn sàng cho production.

LangChain được dẫn dắt bởi một vài niềm tin cốt lõi:

* Large Language Model (LLM) là một công nghệ mới mạnh mẽ, tuyệt vời.
* LLM còn tốt hơn nữa khi được kết hợp với các nguồn dữ liệu bên ngoài.
* LLM sẽ thay đổi hình dạng của các ứng dụng trong tương lai. Cụ thể, ứng dụng tương lai sẽ ngày càng mang tính agentic nhiều hơn.
* Quá trình chuyển đổi này vẫn còn ở giai đoạn rất sớm.
* Dù việc dựng prototype cho các ứng dụng agentic đó khá dễ, việc xây dựng agent đủ tin cậy để đưa vào production vẫn còn rất khó.

Ngày nay, developer có thể chọn cách xây dựng agent: dùng [LangChain](/oss/python/langchain/overview) để có sự linh hoạt và kiểm soát tối đa, hoặc [Deep Agents](/oss/python/langchain/overview), thứ cho phép linh hoạt và kiểm soát tương tự nhưng đi kèm các tool lập kế hoạch (planning), filesystem, subagent, và quản lý context được thiết kế sẵn (opinionated). Cả hai đều được xây dựng trên nền [LangGraph](/oss/python/langgraph/overview).

Với LangChain, chúng tôi có hai trọng tâm chính:

1. **Chúng tôi muốn giúp developer xây dựng được với những model tốt nhất.**
   Các provider khác nhau expose các API khác nhau, với tham số model và định dạng message khác nhau. Chuẩn hóa input/output của model là một trọng tâm cốt lõi, giúp developer dễ dàng chuyển sang model state-of-the-art mới nhất, tránh bị khóa chặt (lock-in) vào một provider.

2. **Chúng tôi muốn việc dùng model để điều phối các luồng phức tạp hơn, tương tác với dữ liệu và tính toán khác, trở nên dễ dàng.**
   Model không nên chỉ dùng để *sinh văn bản* (text generation), mà còn nên được dùng để điều phối các luồng phức tạp hơn, tương tác với dữ liệu khác. LangChain giúp việc định nghĩa [tool](/oss/python/langchain/tools) mà LLM có thể dùng động trở nên dễ dàng, đồng thời hỗ trợ parse và truy cập dữ liệu phi cấu trúc (unstructured).

## Lịch sử

Do tốc độ thay đổi liên tục trong lĩnh vực này, LangChain cũng đã tiến hóa theo thời gian. Dưới đây là dòng thời gian ngắn gọn về cách LangChain thay đổi qua các năm, song song với sự thay đổi trong ý nghĩa của việc "xây dựng với LLM":

**2022-10-24 (v0.0.1)**
Một tháng trước khi ChatGPT ra mắt, **LangChain được ra mắt như một package Python**. Nó gồm hai thành phần chính:

* Các abstraction cho LLM
* "Chain", tức các bước tính toán được xác định trước để chạy cho các use case phổ biến. Ví dụ, RAG: chạy một bước retrieval, rồi chạy một bước generation.

Tên LangChain đến từ "Language" (như trong Language model) và "Chains".

**2022-12**
Các agent đa dụng (general purpose) đầu tiên được thêm vào LangChain.

Các agent đa dụng này dựa trên [bài báo ReAct](https://arxiv.org/abs/2210.03629) (ReAct là viết tắt của Reasoning and Acting). Chúng dùng LLM để sinh JSON đại diện cho tool call, rồi parse JSON đó để xác định tool nào cần gọi.

**2023-01**
OpenAI phát hành API 'Chat Completion'.

Trước đó, model nhận vào string và trả về string. Với ChatCompletions API, model tiến hóa để nhận vào một danh sách message và trả về một message. Các model provider khác cũng làm theo, và LangChain cập nhật để hoạt động với danh sách message.

**2023-01**
LangChain phát hành phiên bản JavaScript.

LLM và agent sẽ thay đổi cách ứng dụng được xây dựng, và JavaScript là ngôn ngữ của các application developer.

**2023-02**
**LangChain Inc. được thành lập như một công ty** xoay quanh dự án open source LangChain.

Mục tiêu chính là "đưa intelligent agent trở nên phổ biến khắp nơi" (ubiquitous). Team nhận ra rằng dù LangChain là một phần quan trọng (LangChain giúp việc bắt đầu với LLM trở nên đơn giản), vẫn cần thêm các thành phần khác.

**2023-03**
OpenAI phát hành 'function calling' trong API của họ.

Điều này cho phép API sinh trực tiếp payload đại diện cho tool call. Các model provider khác cũng làm theo, và LangChain được cập nhật để dùng cách này làm phương pháp ưu tiên cho tool calling (thay vì parse JSON).

**2023-06**
**LangSmith được phát hành** dưới dạng nền tảng closed source bởi LangChain Inc., cung cấp observability và evals (đánh giá).

Vấn đề chính của việc xây dựng agent là làm cho chúng đủ tin cậy, và LangSmith, với khả năng observability và evals, được xây dựng để giải quyết nhu cầu đó. LangChain được cập nhật để tích hợp liền mạch với LangSmith.

**2024-01 (v0.1.0)**
**LangChain phát hành bản 0.1.0**, phiên bản non-0.0.x đầu tiên.

Ngành này đã trưởng thành từ giai đoạn prototype sang production, và vì vậy LangChain tăng trọng tâm vào tính ổn định (stability).

**2024-02**
**LangGraph được phát hành** như một thư viện open source.

LangChain ban đầu có hai trọng tâm: abstraction cho LLM, và interface cấp cao để bắt đầu nhanh với các ứng dụng phổ biến; tuy nhiên, nó còn thiếu một lớp điều phối (orchestration) cấp thấp cho phép developer kiểm soát chính xác luồng chạy của agent. Và LangGraph ra đời.

Khi xây dựng LangGraph, chúng tôi rút kinh nghiệm từ việc xây dựng LangChain và bổ sung các chức năng phát hiện là cần thiết: streaming, durable execution, short-term memory, human-in-the-loop, và nhiều hơn nữa.

**2024-06**
**LangChain có hơn 700 tích hợp (integration).**

Các tích hợp được tách ra khỏi package lõi LangChain, chuyển vào các package độc lập riêng (với các tích hợp lõi) hoặc vào `langchain-community`.

**2024-10**
LangGraph trở thành cách được ưu tiên để xây dựng bất kỳ ứng dụng AI nào phức tạp hơn một lệnh gọi LLM đơn lẻ.

Khi developer cố gắng cải thiện độ tin cậy của ứng dụng, họ cần nhiều quyền kiểm soát hơn những gì interface cấp cao cung cấp. LangGraph cung cấp sự linh hoạt cấp thấp đó. Hầu hết chain và agent trong LangChain được đánh dấu deprecated kèm hướng dẫn migrate sang LangGraph. Vẫn còn một abstraction cấp cao được tạo trong LangGraph: agent abstraction. Nó được xây dựng trên nền LangGraph cấp thấp và có cùng interface với ReAct agent từ LangChain.

**2025-04**
API của model trở nên đa phương thức (multimodal) hơn.

Model bắt đầu chấp nhận file, ảnh, video, và nhiều hơn nữa. Chúng tôi cập nhật định dạng message của `langchain-core` tương ứng, để cho phép developer chỉ định các input đa phương thức theo cách chuẩn hóa.

**2025-10-20 (v1.0.0)**
**LangChain phát hành bản 1.0** với hai thay đổi lớn:

1. Đại tu toàn bộ chain và agent trong `langchain`. Toàn bộ chain và agent giờ được thay thế bằng duy nhất một abstraction cấp cao: agent abstraction được xây dựng trên nền LangGraph. Đây chính là abstraction cấp cao ban đầu được tạo ra trong LangGraph, nay chỉ đơn giản được chuyển sang LangChain.

   Với người dùng vẫn đang dùng chain/agent cũ của LangChain và KHÔNG muốn nâng cấp (lưu ý: chúng tôi khuyến nghị bạn nên nâng cấp), bạn có thể tiếp tục dùng LangChain cũ bằng cách cài package `langchain-classic`.

2. Một định dạng nội dung message chuẩn hóa: API của model đã tiến hóa từ việc trả về message với một content string đơn giản sang các kiểu output phức tạp hơn, như khối reasoning, trích dẫn (citation), tool call phía server, v.v. LangChain tiến hóa định dạng message của mình để chuẩn hóa các kiểu này trên nhiều provider.

**2026-03-15 (v0.5.3)**
**Deep Agents được phát hành** như một agent harness open source xây dựng trên nền LangGraph.

Trong khi LangChain cung cấp các building block linh hoạt cho kiến trúc agent tùy chỉnh, [Deep Agents](/oss/python/langchain/overview) cung cấp một lựa chọn "đầy đủ tính năng" (batteries-included) cho các tác vụ phức tạp, chạy dài như research và coding. Nó bổ sung các tool lập kế hoạch (planning) có sẵn, một virtual filesystem với backend có thể cắm thay thế (pluggable, gồm in-memory, disk, LangGraph store, sandbox), và khả năng sinh subagent để cô lập context. Dùng Deep Agents cho các agent tự trị (autonomous) hơn với tool được định nghĩa sẵn; dùng LangChain khi cần kiểm soát toàn diện kiến trúc agent của bạn.
