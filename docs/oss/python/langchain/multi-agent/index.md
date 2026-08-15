# Đa agent (Multi-agent)

Các hệ thống multi-agent (đa agent) điều phối các thành phần chuyên biệt để xử lý các workflow phức tạp. Tuy nhiên, không phải tác vụ phức tạp nào cũng cần đến cách tiếp cận này: một agent duy nhất với bộ tool và prompt phù hợp (đôi khi mang tính động) thường có thể đạt kết quả tương tự.

!!! tip "Mẹo"
    Để có hỗ trợ multi-agent tích hợp sẵn, hãy dùng [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview): một harness cấp cao hơn được xây dựng trên LangChain, đi kèm sẵn [subagent](https://docs.langchain.com/oss/python/deepagents/subagents), [skill](https://docs.langchain.com/oss/python/deepagents/skills), khả năng lập kế hoạch (planning), một virtual filesystem, và quản lý context.

## Tại sao cần multi-agent?

Khi các nhà phát triển nói rằng họ cần "multi-agent", thường họ đang tìm kiếm một hoặc nhiều khả năng sau:

* **Quản lý context (Context management)**: Cung cấp kiến thức chuyên biệt mà không làm quá tải context window của model. Nếu context là vô hạn và độ trễ bằng 0, bạn có thể nhồi toàn bộ kiến thức vào một prompt duy nhất, nhưng vì thực tế không phải vậy, bạn cần các pattern để chọn lọc và hiển thị thông tin liên quan.
* **Phát triển phân tán (Distributed development)**: Cho phép các team khác nhau phát triển và duy trì các khả năng một cách độc lập, sau đó kết hợp chúng thành một hệ thống lớn hơn với ranh giới rõ ràng.
* **Song song hoá (Parallelization)**: Sinh ra các worker chuyên biệt cho từng subtask và thực thi chúng đồng thời để có kết quả nhanh hơn.

Các pattern multi-agent đặc biệt hữu ích khi một agent duy nhất có quá nhiều [tool](https://docs.langchain.com/oss/python/langchain/tools) và đưa ra quyết định kém về việc nên dùng tool nào, khi tác vụ đòi hỏi kiến thức chuyên biệt với context lớn (prompt dài và tool đặc thù theo domain), hoặc khi bạn cần áp đặt các ràng buộc tuần tự chỉ mở khoá một số khả năng sau khi đáp ứng điều kiện nhất định.

!!! tip "Mẹo"
    Trọng tâm của việc thiết kế multi-agent là **[context engineering](https://docs.langchain.com/oss/python/langchain/context-engineering)**: quyết định thông tin nào mỗi agent được thấy. Chất lượng hệ thống của bạn phụ thuộc vào việc đảm bảo mỗi agent có quyền truy cập đúng dữ liệu cần thiết cho tác vụ của nó.

## Các pattern

Dưới đây là các pattern chính để xây dựng hệ thống multi-agent, mỗi pattern phù hợp với các use case khác nhau:

| Pattern | Cách hoạt động |
| --- | --- |
| [**Subagents**](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents) | Một agent chính điều phối các subagent như những tool. Toàn bộ việc định tuyến đều đi qua agent chính, agent này quyết định khi nào và cách nào để gọi từng subagent. |
| [**Handoffs**](handoffs.md) | Hành vi thay đổi linh động dựa trên state. Các tool call cập nhật một biến state, từ đó kích hoạt việc định tuyến hoặc thay đổi cấu hình, chuyển đổi agent hoặc điều chỉnh tool và prompt của agent hiện tại. |
| [**Skills**](skills.md) | Các prompt và kiến thức chuyên biệt được tải theo yêu cầu (on-demand). Một agent duy nhất vẫn giữ quyền kiểm soát trong khi tải context từ các skill khi cần. |
| [**Router**](router.md) | Một bước định tuyến (routing) phân loại đầu vào và chuyển nó tới một hoặc nhiều agent chuyên biệt. Kết quả được tổng hợp thành một câu trả lời chung. |
| [**Custom workflow**](custom-workflow.md) | Xây dựng luồng thực thi tuỳ biến với [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview), kết hợp logic tất định (deterministic) và hành vi agentic. Nhúng các pattern khác dưới dạng node trong workflow của bạn. |

### Chọn pattern phù hợp

Dùng bảng dưới đây để đối chiếu yêu cầu của bạn với pattern phù hợp:

| Pattern | Phát triển phân tán | Song song hoá | Multi-hop | Tương tác trực tiếp với người dùng |
| --- | :---: | :---: | :---: | :---: |
| [**Subagents**](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ |
| [**Handoffs**](handoffs.md) | - | - | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| [**Skills**](skills.md) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| [**Router**](router.md) | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | - | ⭐⭐⭐ |

* **Phát triển phân tán**: Các team khác nhau có thể duy trì các thành phần một cách độc lập không?
* **Song song hoá**: Nhiều agent có thể thực thi đồng thời không?
* **Multi-hop**: Pattern này có hỗ trợ gọi nhiều subagent tuần tự (nối tiếp) không?
* **Tương tác trực tiếp với người dùng**: Các subagent có thể trò chuyện trực tiếp với người dùng không?

!!! tip "Mẹo"
    Bạn có thể kết hợp nhiều pattern với nhau! Ví dụ, một kiến trúc **subagents** có thể gọi các tool mà bản thân các tool đó lại gọi custom workflow hoặc router agent. Các subagent thậm chí có thể dùng pattern **skills** để tải context theo yêu cầu. Khả năng kết hợp gần như là vô hạn!

### Tổng quan trực quan

=== "Subagents"

    Một agent chính điều phối các subagent như những tool. Toàn bộ việc định tuyến đều đi qua agent chính.

    <img src="https://mintcdn.com/langchain-5e9cc07a/CRpSg52QqwDx49Bw/oss/langchain/multi-agent/images/pattern-subagents.png?fit=max&auto=format&n=CRpSg52QqwDx49Bw&q=85&s=f924dde09057820b08f0c577e08fcfe7" alt="Subagents pattern: main agent coordinates subagents as tools" width="1020" height="734" />

=== "Handoffs"

    Các agent chuyển giao quyền điều khiển cho nhau thông qua tool call. Mỗi agent có thể handoff sang agent khác hoặc trả lời trực tiếp cho người dùng.

    <img src="https://mintcdn.com/langchain-5e9cc07a/CRpSg52QqwDx49Bw/oss/langchain/multi-agent/images/pattern-handoffs.png?fit=max&auto=format&n=CRpSg52QqwDx49Bw&q=85&s=57d935e6a8efab4afb3faa385113f4dd" alt="Handoffs pattern: agents transfer control via tool calls" width="1568" height="464" />

=== "Skills"

    Một agent duy nhất tải các prompt và kiến thức chuyên biệt theo yêu cầu trong khi vẫn giữ quyền kiểm soát.

    <img src="https://mintcdn.com/langchain-5e9cc07a/CRpSg52QqwDx49Bw/oss/langchain/multi-agent/images/pattern-skills.png?fit=max&auto=format&n=CRpSg52QqwDx49Bw&q=85&s=119131d1f19be1f0c6fb1e30f080b427" alt="Skills pattern: single agent loads specialized context on-demand" width="874" height="734" />

=== "Router"

    Một bước định tuyến phân loại đầu vào và chuyển tới các agent chuyên biệt. Kết quả được tổng hợp lại.

    <img src="https://mintcdn.com/langchain-5e9cc07a/CRpSg52QqwDx49Bw/oss/langchain/multi-agent/images/pattern-router.png?fit=max&auto=format&n=CRpSg52QqwDx49Bw&q=85&s=ceab32819240ba87f3a132357cc78b09" alt="Router pattern: routing step classifies input to specialized agents" width="1560" height="556" />

!!! tip "Mẹo"
    Theo dõi (trace) toàn bộ luồng phối hợp giữa các agent với [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-multi-agent-index). Làm theo [hướng dẫn nhanh về tracing](https://docs.langchain.com/langsmith/trace-with-langchain) để thiết lập.

    Chúng tôi cũng khuyến nghị bạn thiết lập [LangSmith Engine](https://docs.langchain.com/langsmith/engine), công cụ này giám sát các trace của bạn, phát hiện vấn đề và đề xuất cách khắc phục.

## So sánh hiệu năng

Mỗi pattern có đặc điểm hiệu năng khác nhau. Hiểu rõ những đánh đổi này giúp bạn chọn đúng pattern phù hợp với yêu cầu về độ trễ và chi phí.

**Các chỉ số chính:**

* **Model calls**: Số lượt gọi LLM. Càng nhiều lượt gọi thì độ trễ càng cao (đặc biệt nếu tuần tự) và chi phí API trên mỗi request càng lớn.
* **Tokens processed**: Tổng mức sử dụng [context window](https://docs.langchain.com/oss/python/langchain/context-engineering) qua tất cả các lượt gọi. Càng nhiều token thì chi phí xử lý càng cao và có khả năng chạm giới hạn context.

### Yêu cầu một lượt (one-shot)

> **Người dùng:** "Mua cà phê"

Một agent/skill chuyên biệt về cà phê có thể gọi tool `buy_coffee`.

| Pattern | Số lượt gọi model | Phù hợp nhất |
| --- | :---: | :---: |
| [**Subagents**](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents) | 4 | |
| [**Handoffs**](handoffs.md) | 3 | ✅ |
| [**Skills**](skills.md) | 3 | ✅ |
| [**Router**](router.md) | 3 | ✅ |

=== "Subagents"

    **4 lượt gọi model:**

    <img src="https://mintcdn.com/langchain-5e9cc07a/CRpSg52QqwDx49Bw/oss/langchain/multi-agent/images/oneshot-subagents.png?fit=max&auto=format&n=CRpSg52QqwDx49Bw&q=85&s=bd4eeef41d8870bfa887dd0aa97d0b79" alt="Subagents one-shot: 4 model calls for buy coffee request" width="1568" height="1124" />

=== "Handoffs"

    **3 lượt gọi model:**

    <img src="https://mintcdn.com/langchain-5e9cc07a/CRpSg52QqwDx49Bw/oss/langchain/multi-agent/images/oneshot-handoffs.png?fit=max&auto=format&n=CRpSg52QqwDx49Bw&q=85&s=42ec50519ff04f034050dc77cf869907" alt="Handoffs one-shot: 3 model calls for buy coffee request" width="1568" height="948" />

=== "Skills"

    **3 lượt gọi model:**

    <img src="https://mintcdn.com/langchain-5e9cc07a/CRpSg52QqwDx49Bw/oss/langchain/multi-agent/images/oneshot-skills.png?fit=max&auto=format&n=CRpSg52QqwDx49Bw&q=85&s=c8dbf69ed4509e30e5280e7e8a391dab" alt="Skills one-shot: 3 model calls for buy coffee request" width="1568" height="1036" />

=== "Router"

    **3 lượt gọi model:**

    <img src="https://mintcdn.com/langchain-5e9cc07a/CRpSg52QqwDx49Bw/oss/langchain/multi-agent/images/oneshot-router.png?fit=max&auto=format&n=CRpSg52QqwDx49Bw&q=85&s=be5707931d3e520e3ae66af544f2cf2f" alt="Router one-shot: 3 model calls for buy coffee request" width="1568" height="994" />

**Nhận định chính:** Handoffs, Skills và Router hiệu quả nhất cho các tác vụ đơn lẻ (mỗi pattern chỉ 3 lượt gọi). Subagents tốn thêm một lượt gọi vì kết quả phải chảy ngược qua agent chính; chi phí phát sinh này đổi lại quyền kiểm soát tập trung.

### Yêu cầu lặp lại

> **Lượt 1:** "Mua cà phê"
> **Lượt 2:** "Mua cà phê lần nữa"

Người dùng lặp lại cùng một yêu cầu trong cùng một cuộc hội thoại.

| Pattern | Số lượt gọi ở lượt 2 | Tổng (cả hai lượt) | Phù hợp nhất |
| --- | :---: | :---: | :---: |
| [**Subagents**](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents) | 4 | 8 | |
| [**Handoffs**](handoffs.md) | 2 | 5 | ✅ |
| [**Skills**](skills.md) | 2 | 5 | ✅ |
| [**Router**](router.md) | 3 | 6 | |

=== "Subagents"

    **4 lượt gọi lại → tổng 8 lượt**

    * Subagent được thiết kế **không lưu trạng thái (stateless)**: mỗi lần gọi đều theo cùng một luồng
    * Agent chính duy trì context hội thoại, nhưng các subagent luôn bắt đầu lại từ đầu mỗi lần
    * Điều này mang lại khả năng cô lập context mạnh nhưng lặp lại toàn bộ luồng xử lý

=== "Handoffs"

    **2 lượt gọi → tổng 5 lượt**

    * Agent cà phê **vẫn đang hoạt động** từ lượt 1 (state được giữ lại)
    * Không cần handoff: agent gọi trực tiếp tool `buy_coffee` (lượt gọi 1)
    * Agent trả lời người dùng (lượt gọi 2)
    * **Tiết kiệm 1 lượt gọi nhờ bỏ qua bước handoff**

=== "Skills"

    **2 lượt gọi → tổng 5 lượt**

    * Context của skill **đã được tải sẵn** trong lịch sử hội thoại
    * Không cần tải lại: agent gọi trực tiếp tool `buy_coffee` (lượt gọi 1)
    * Agent trả lời người dùng (lượt gọi 2)
    * **Tiết kiệm 1 lượt gọi nhờ tái sử dụng skill đã tải**

=== "Router"

    **3 lượt gọi lại → tổng 6 lượt**

    * Router **không lưu trạng thái**: mỗi request đều cần một lượt gọi LLM để định tuyến
    * Lượt 2: lượt gọi LLM của router (1) → agent Milk gọi buy_coffee (2) → agent Milk trả lời (3)
    * Có thể tối ưu bằng cách bọc router thành một tool trong một agent có trạng thái (stateful)

**Nhận định chính:** Các pattern có trạng thái (Handoffs, Skills) tiết kiệm 40-50% số lượt gọi khi có yêu cầu lặp lại. Subagents duy trì chi phí ổn định cho mỗi request; thiết kế không lưu trạng thái này mang lại khả năng cô lập context mạnh nhưng phải đánh đổi bằng việc lặp lại các lượt gọi model.

### Đa lĩnh vực (multi-domain)

> **Người dùng:** "So sánh Python, JavaScript và Rust cho phát triển web"

Mỗi agent/skill ngôn ngữ chứa khoảng 2000 token tài liệu. Tất cả các pattern đều có thể thực hiện tool call song song.

| Pattern | Số lượt gọi model | Tổng số token | Phù hợp nhất |
| --- | :---: | :---: | :---: |
| [**Subagents**](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents) | 5 | ~9K | ✅ |
| [**Handoffs**](handoffs.md) | 7+ | ~14K+ | |
| [**Skills**](skills.md) | 3 | ~15K | |
| [**Router**](router.md) | 5 | ~9K | ✅ |

=== "Subagents"

    **5 lượt gọi, ~9K token**

    <img src="https://mintcdn.com/langchain-5e9cc07a/CRpSg52QqwDx49Bw/oss/langchain/multi-agent/images/multidomain-subagents.png?fit=max&auto=format&n=CRpSg52QqwDx49Bw&q=85&s=9cc5d2d46bfa98b7ceeacdc473512c94" alt="Subagents multi-domain: 5 calls with parallel execution" width="1568" height="1232" />

    Mỗi subagent hoạt động **cô lập (isolation)** chỉ với context liên quan tới nó. Tổng: **9K token**.

=== "Handoffs"

    **7+ lượt gọi, ~14K+ token**

    <img src="https://mintcdn.com/langchain-5e9cc07a/CRpSg52QqwDx49Bw/oss/langchain/multi-agent/images/multidomain-handoffs.png?fit=max&auto=format&n=CRpSg52QqwDx49Bw&q=85&s=7ede44260515e49ff1d0217f0030d66d" alt="Handoffs multi-domain: 7+ sequential calls" width="1568" height="834" />

    Handoffs thực thi **tuần tự**: không thể nghiên cứu cả ba ngôn ngữ song song. Lịch sử hội thoại ngày càng dài làm tăng chi phí phát sinh. Tổng: **~14K+ token**.

=== "Skills"

    **3 lượt gọi, ~15K token**

    <img src="https://mintcdn.com/langchain-5e9cc07a/CRpSg52QqwDx49Bw/oss/langchain/multi-agent/images/multidomain-skills.png?fit=max&auto=format&n=CRpSg52QqwDx49Bw&q=85&s=2162584b6076aee83396760bc6de4cf4" alt="Skills multi-domain: 3 calls with accumulated context" width="1560" height="988" />

    Sau khi tải, **mọi lượt gọi tiếp theo đều phải xử lý toàn bộ 6K token tài liệu skill**. Subagents xử lý ít hơn 67% tổng số token nhờ việc cô lập context. Tổng: **15K token**.

=== "Router"

    **5 lượt gọi, ~9K token**

    <img src="https://mintcdn.com/langchain-5e9cc07a/CRpSg52QqwDx49Bw/oss/langchain/multi-agent/images/multidomain-router.png?fit=max&auto=format&n=CRpSg52QqwDx49Bw&q=85&s=ef11573bc65e5a2996d671bb3030ca6b" alt="Router multi-domain: 5 calls with parallel execution" width="1568" height="1052" />

    Router dùng **LLM để định tuyến**, sau đó gọi các agent song song. Tương tự Subagents nhưng có thêm bước định tuyến rõ ràng. Tổng: **9K token**.

**Nhận định chính:** Đối với các tác vụ đa lĩnh vực, các pattern có khả năng thực thi song song (Subagents, Router) hiệu quả nhất. Skills có ít lượt gọi hơn nhưng tốn nhiều token do context tích luỹ dần. Handoffs kém hiệu quả trong trường hợp này: nó buộc phải thực thi tuần tự và không thể tận dụng tool calling song song để tra cứu nhiều lĩnh vực cùng lúc.

### Tổng kết

Dưới đây là cách các pattern so sánh với nhau trên cả ba kịch bản:

| Pattern | Một lượt (one-shot) | Yêu cầu lặp lại | Đa lĩnh vực |
| --- | :---: | :---: | :---: |
| [**Subagents**](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents) | 4 lượt gọi | 8 lượt gọi (4+4) | 5 lượt gọi, 9K token |
| [**Handoffs**](handoffs.md) | 3 lượt gọi | 5 lượt gọi (3+2) | 7+ lượt gọi, 14K+ token |
| [**Skills**](skills.md) | 3 lượt gọi | 5 lượt gọi (3+2) | 3 lượt gọi, 15K token |
| [**Router**](router.md) | 3 lượt gọi | 6 lượt gọi (3+3) | 5 lượt gọi, 9K token |

**Chọn pattern:**

| Tối ưu cho | [Subagents](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents) | [Handoffs](handoffs.md) | [Skills](skills.md) | [Router](router.md) |
| --- | :---: | :---: | :---: | :---: |
| Yêu cầu đơn lẻ | | ✅ | ✅ | ✅ |
| Yêu cầu lặp lại | | ✅ | ✅ | |
| Thực thi song song | ✅ | | | ✅ |
| Lĩnh vực có context lớn | ✅ | | | ✅ |
| Tác vụ đơn giản, tập trung | | | ✅ | |

---

[Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode và nhiều công cụ khác qua MCP để nhận câu trả lời theo thời gian thực.

[Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/multi-agent/index.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
