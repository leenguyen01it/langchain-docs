# Tổng quan LangGraph

> Giành quyền kiểm soát với LangGraph để thiết kế agent xử lý tin cậy các tác vụ phức tạp

Được các công ty đang định hình tương lai của agent tin dùng, bao gồm Klarna, Uber, J.P. Morgan và nhiều công ty khác, LangGraph là một framework điều phối (orchestration) cấp thấp và runtime để xây dựng, quản lý, và triển khai các agent có trạng thái (stateful), chạy dài (long-running). LangGraph cho bạn quyền kiểm soát chi tiết để trộn các bước tất định (deterministic), viết tay với các bước agentic do LLM điều khiển trong cùng một graph, nhờ đó bạn có thể xây dựng agent tùy chỉnh hoạt động đúng theo cách ứng dụng của bạn yêu cầu.

LangGraph rất cấp thấp, và tập trung hoàn toàn vào việc **điều phối (orchestration)** agent. Trước khi dùng LangGraph, chúng tôi khuyến nghị bạn làm quen với một số thành phần dùng để xây dựng agent, bắt đầu từ [model](../langchain/models.md) và [tool](../langchain/tools.md).

Chúng tôi sẽ thường xuyên dùng các thành phần [LangChain](../langchain/overview.md) xuyên suốt tài liệu để tích hợp model và tool, nhưng bạn không cần dùng LangChain để dùng LangGraph. Nếu bạn mới bắt đầu với agent hoặc muốn một tầng trừu tượng cấp cao hơn, chúng tôi khuyến nghị bạn dùng [agent](../langchain/agents.md) của LangChain, thứ cung cấp sẵn các kiến trúc dựng sẵn cho các vòng lặp model và tool-calling phổ biến.

LangGraph tập trung vào các khả năng nền tảng quan trọng cho việc điều phối agent: thực thi bền vững (durable execution), streaming, human-in-the-loop, và nhiều hơn nữa.

Một trong những điểm mạnh cốt lõi của LangGraph là khả năng trộn các bước tất định với các bước agentic do LLM điều khiển trong một graph duy nhất. Điều này cho phép bạn xây dựng workflow tùy chỉnh, nơi một phần logic hoàn toàn có thể dự đoán và kiểm tra được (auditable) trong khi các phần khác linh hoạt và do model dẫn dắt, cho bạn quyền kiểm soát chi tiết về chính xác nơi và cách AI được áp dụng.

??? note "Các sản phẩm LangChain khớp với nhau như thế nào"
    * [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview) là một [agent harness](https://docs.langchain.com/oss/python/concepts/products#agent-harnesses-like-the-deep-agents-sdk): lập kế hoạch, subagent, tool filesystem, và quản lý context trên nền LangGraph.
    * [LangChain](../langchain/overview.md) là agent framework: các tầng trừu tượng và tích hợp cho model, tool, và vòng lặp agent.
    * [LangGraph](overview.md) là orchestration runtime: thực thi bền vững, streaming, human-in-the-loop, và persistence.
    * [LangSmith](https://docs.langchain.com/langsmith/observability) là nền tảng để trace, đánh giá (evaluation), prompt, và deploy trên nhiều framework.
    * [LangSmith Engine](https://docs.langchain.com/langsmith/engine) phát hiện vấn đề trong trace agent LangGraph của bạn và đề xuất cách khắc phục. Bạn có thể mở một pull request với bản sửa được đề xuất trực tiếp từ tab Engine.
    * [LangSmith Fleet](https://docs.langchain.com/langsmith/fleet/index) là công cụ xây dựng agent no-code cho template, tích hợp, và tự động hóa tác vụ định kỳ.

    Đọc [Framework, runtime, và harness](https://docs.langchain.com/oss/python/concepts/products) để so sánh các stack mã nguồn mở.

## Cài đặt

=== "pip"
    ```bash
    pip install -U langgraph
    ```

=== "uv"
    ```bash
    uv add langgraph
    ```

Sau đó, tạo một ví dụ hello world đơn giản:

```python
from langgraph.graph import StateGraph, MessagesState, START, END

def mock_llm(state: MessagesState):
    return {"messages": [{"role": "ai", "content": "hello world"}]}

graph = StateGraph(MessagesState)
graph.add_node(mock_llm)
graph.add_edge(START, "mock_llm")
graph.add_edge("mock_llm", END)
graph = graph.compile()

graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
```

!!! tip
    Dùng [LangSmith](https://docs.langchain.com/langsmith/observability) để trace request, debug hành vi agent, và đánh giá output. Set `LANGSMITH_TRACING=true` cùng API key để bắt đầu. Làm theo [tracing quickstart](https://docs.langchain.com/langsmith/trace-with-langchain) để thiết lập. Chúng tôi cũng khuyến nghị bạn thiết lập [LangSmith Engine](https://docs.langchain.com/langsmith/engine), công cụ giám sát trace, phát hiện vấn đề, và đề xuất cách khắc phục.

## Lợi ích cốt lõi

LangGraph cung cấp hạ tầng hỗ trợ cấp thấp cho *bất kỳ* workflow hoặc agent có trạng thái, chạy dài nào. LangGraph không trừu tượng hóa prompt hay kiến trúc, và cung cấp các lợi ích trung tâm sau:

* **Trộn các bước tất định và agentic**: Kết hợp logic tất định, viết tay với việc ra quyết định do LLM điều khiển trong một graph duy nhất. Dùng bước tất định khi cần độ tin cậy và khả năng dự đoán, và bước agentic khi cần sự linh hoạt, cho bạn quyền kiểm soát chính xác từng phần trong hành vi của agent.
* [Persistence](persistence.md): Xây dựng agent chịu được lỗi và có thể chạy trong thời gian dài, tiếp tục từ nơi đã dừng lại.
* [Human-in-the-loop](interrupts.md): Kết hợp giám sát của con người bằng cách kiểm tra và sửa đổi trạng thái agent tại bất kỳ thời điểm nào.
* [Bộ nhớ toàn diện](https://docs.langchain.com/oss/python/concepts/memory): Tạo agent có trạng thái với cả bộ nhớ làm việc ngắn hạn cho việc suy luận đang diễn ra và bộ nhớ dài hạn xuyên suốt các phiên.
* [Debug với LangSmith](https://docs.langchain.com/langsmith/observability): Có được khả năng quan sát sâu vào hành vi agent phức tạp với công cụ trực quan hóa giúp trace đường thực thi, ghi lại chuyển trạng thái, và cung cấp số liệu runtime chi tiết.
* [Deploy sẵn sàng cho production](https://docs.langchain.com/langsmith/deployment): Deploy các hệ thống agent phức tạp một cách tự tin với hạ tầng có khả năng mở rộng, được thiết kế để xử lý các thách thức đặc thù của workflow có trạng thái, chạy dài.

## Hệ sinh thái LangGraph

Mặc dù LangGraph có thể dùng độc lập, nó cũng tích hợp liền mạch với bất kỳ sản phẩm LangChain nào, mang lại cho developer một bộ công cụ đầy đủ để xây dựng agent. Để cải thiện việc phát triển ứng dụng LLM, kết hợp LangGraph với:

**LangSmith Observability** ([tìm hiểu thêm](https://docs.langchain.com/langsmith/observability))
Trace request, đánh giá output, và giám sát các bản deploy tại một nơi duy nhất. Tạo prototype cục bộ với LangGraph, sau đó chuyển sang production với observability và đánh giá tích hợp để xây dựng hệ thống agent tin cậy hơn.

**LangSmith Deployment** ([tìm hiểu thêm](https://docs.langchain.com/langsmith/deployment))
Deploy và mở rộng agent một cách dễ dàng với một nền tảng deploy được xây dựng riêng cho workflow có trạng thái, chạy dài. Khám phá, tái sử dụng, cấu hình, và chia sẻ agent giữa các nhóm, đồng thời lặp nhanh với việc tạo prototype trực quan trong Studio.

**LangChain** ([tìm hiểu thêm](../langchain/overview.md))
Cung cấp tích hợp và các thành phần có thể kết hợp (composable) để tinh gọn việc phát triển ứng dụng LLM. Chứa các tầng trừu tượng agent được xây dựng trên nền LangGraph.

## Lời cảm ơn

LangGraph lấy cảm hứng từ [Pregel](https://research.google/pubs/pub37252/) và [Apache Beam](https://beam.apache.org/). Giao diện công khai lấy cảm hứng từ [NetworkX](https://networkx.org/documentation/latest/). LangGraph được xây dựng bởi LangChain Inc, đội ngũ tạo ra LangChain, nhưng có thể dùng độc lập không cần LangChain.

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/overview.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
