# Chọn giữa Graph API và Functional API

LangGraph cung cấp hai API khác nhau để xây dựng agent workflow: **Graph API** và **Functional API**. Cả hai API dùng chung một runtime bên dưới và có thể dùng cùng nhau trong cùng một ứng dụng, nhưng chúng được thiết kế cho các trường hợp sử dụng và sở thích phát triển khác nhau.

Hướng dẫn này sẽ giúp bạn hiểu khi nào nên dùng API nào dựa trên yêu cầu cụ thể của bạn.

## Hướng dẫn quyết định nhanh

Dùng **Graph API** khi bạn cần:

* **Trực quan hoá workflow phức tạp** để debug và làm tài liệu
* **Quản lý state tường minh** với dữ liệu chia sẻ giữa nhiều node
* **Phân nhánh có điều kiện** với nhiều điểm quyết định
* **Các nhánh thực thi song song** cần hợp nhất lại sau đó
* **Cộng tác nhóm** khi việc biểu diễn trực quan giúp hiểu dễ hơn

Dùng **Functional API** khi bạn muốn:

* **Thay đổi code tối thiểu** với code thủ tục (procedural) hiện có
* **Control flow tiêu chuẩn** (if/else, vòng lặp, gọi hàm)
* **State giới hạn trong phạm vi hàm** mà không cần quản lý state tường minh
* **Prototype nhanh** với ít boilerplate
* **Workflow tuyến tính** với logic phân nhánh đơn giản

## So sánh chi tiết

### Khi nào dùng Graph API

[Graph API](./graph-api.md) dùng cách tiếp cận khai báo (declarative), trong đó bạn định nghĩa node, cạnh (edge), và state chia sẻ để tạo ra một cấu trúc graph trực quan.

**1. Cây quyết định và logic phân nhánh phức tạp**

Khi workflow của bạn có nhiều điểm quyết định phụ thuộc vào các điều kiện khác nhau, Graph API giúp các nhánh này tường minh và dễ trực quan hoá.

```python
# Graph API: Trực quan hoá rõ ràng các đường quyết định
from langgraph.graph import StateGraph
from typing import TypedDict

class AgentState(TypedDict):
    messages: list
    current_tool: str
    retry_count: int

def should_continue(state):
    if state["retry_count"] > 3:
        return "end"
    elif state["current_tool"] == "search":
        return "process_search"
    else:
        return "call_llm"

workflow = StateGraph(AgentState)
workflow.add_node("call_llm", call_llm_node)
workflow.add_node("process_search", search_node)
workflow.add_conditional_edges("call_llm", should_continue)
```

**2. Quản lý state qua nhiều thành phần**

Khi bạn cần chia sẻ và điều phối state giữa các phần khác nhau trong workflow, việc quản lý state tường minh của Graph API rất hữu ích.

```python
# Nhiều node có thể truy cập và sửa đổi state chung
class WorkflowState(TypedDict):
    user_input: str
    search_results: list
    generated_response: str
    validation_status: str

def search_node(state):
    # Truy cập state chung
    results = search(state["user_input"])
    return {"search_results": results}

def validation_node(state):
    # Truy cập kết quả từ node trước
    is_valid = validate(state["generated_response"])
    return {"validation_status": "valid" if is_valid else "invalid"}
```

**3. Xử lý song song có đồng bộ hoá**

Khi bạn cần chạy nhiều thao tác song song rồi hợp nhất kết quả, Graph API xử lý việc này một cách tự nhiên.

```python
# Xử lý song song nhiều nguồn dữ liệu
workflow.add_node("fetch_news", fetch_news)
workflow.add_node("fetch_weather", fetch_weather)
workflow.add_node("fetch_stocks", fetch_stocks)
workflow.add_node("combine_data", combine_all_data)

# Tất cả thao tác fetch chạy song song
workflow.add_edge(START, "fetch_news")
workflow.add_edge(START, "fetch_weather")
workflow.add_edge(START, "fetch_stocks")

# combine_data chờ tất cả thao tác song song hoàn tất
workflow.add_edge("fetch_news", "combine_data")
workflow.add_edge("fetch_weather", "combine_data")
workflow.add_edge("fetch_stocks", "combine_data")
```

**4. Phát triển nhóm và tài liệu hoá**

Bản chất trực quan của Graph API giúp các nhóm dễ hiểu, làm tài liệu, và bảo trì các workflow phức tạp hơn.

```python
# Phân tách rõ ràng trách nhiệm - mỗi thành viên nhóm có thể làm việc trên các node khác nhau
workflow.add_node("data_ingestion", data_team_function)
workflow.add_node("ml_processing", ml_team_function)
workflow.add_node("business_logic", product_team_function)
workflow.add_node("output_formatting", frontend_team_function)
```

### Khi nào dùng Functional API

[Functional API](./functional-api.md) dùng cách tiếp cận mệnh lệnh (imperative), tích hợp các tính năng của LangGraph vào code thủ tục tiêu chuẩn.

**1. Code thủ tục có sẵn**

Khi bạn có code hiện có dùng control flow tiêu chuẩn và muốn thêm tính năng LangGraph với việc refactor tối thiểu.

```python
# Functional API: Thay đổi tối thiểu với code hiện có
from langgraph.func import entrypoint, task

@task
def process_user_input(user_input: str) -> dict:
    # Hàm hiện có với thay đổi tối thiểu
    return {"processed": user_input.lower().strip()}

@entrypoint(checkpointer=checkpointer)
def workflow(user_input: str) -> str:
    # Control flow Python tiêu chuẩn
    processed = process_user_input(user_input).result()

    if "urgent" in processed["processed"]:
        response = handle_urgent_request(processed).result()
    else:
        response = handle_normal_request(processed).result()

    return response
```

**2. Workflow tuyến tính với logic đơn giản**

Khi workflow của bạn chủ yếu tuần tự với logic điều kiện đơn giản.

```python
@entrypoint(checkpointer=checkpointer)
def essay_workflow(topic: str) -> dict:
    # Luồng tuyến tính với phân nhánh đơn giản
    outline = create_outline(topic).result()

    if len(outline["points"]) < 3:
        outline = expand_outline(outline).result()

    draft = write_draft(outline).result()

    # Điểm dừng để con người review
    feedback = interrupt({"draft": draft, "action": "Please review"})

    if feedback == "approve":
        final_essay = draft
    else:
        final_essay = revise_essay(draft, feedback).result()

    return {"essay": final_essay}
```

**3. Prototype nhanh**

Khi bạn muốn thử nghiệm ý tưởng nhanh chóng mà không cần định nghĩa state schema và cấu trúc graph.

```python
@entrypoint(checkpointer=checkpointer)
def quick_prototype(data: dict) -> dict:
    # Lặp nhanh - không cần state schema
    step1_result = process_step1(data).result()
    step2_result = process_step2(step1_result).result()

    return {"final_result": step2_result}
```

**4. Quản lý state trong phạm vi hàm**

Khi state của bạn tự nhiên nằm trong phạm vi từng hàm riêng lẻ và không cần chia sẻ rộng rãi.

```python
@task
def analyze_document(document: str) -> dict:
    # Quản lý state cục bộ trong hàm
    sections = extract_sections(document)
    summaries = [summarize(section) for section in sections]
    key_points = extract_key_points(summaries)

    return {
        "sections": len(sections),
        "summaries": summaries,
        "key_points": key_points
    }

@entrypoint(checkpointer=checkpointer)
def document_processor(document: str) -> dict:
    analysis = analyze_document(document).result()
    # State được truyền giữa các hàm khi cần
    return generate_report(analysis).result()
```

## Kết hợp cả hai API

Bạn có thể dùng cả hai API cùng nhau trong cùng một ứng dụng. Điều này hữu ích khi các phần khác nhau của hệ thống có yêu cầu khác nhau.

```python
from langgraph.graph import StateGraph
from langgraph.func import entrypoint

# Điều phối multi-agent phức tạp dùng Graph API
coordination_graph = StateGraph(CoordinationState)
coordination_graph.add_node("orchestrator", orchestrator_node)
coordination_graph.add_node("agent_a", agent_a_node)
coordination_graph.add_node("agent_b", agent_b_node)

# Xử lý dữ liệu đơn giản dùng Functional API
@entrypoint()
def data_processor(raw_data: dict) -> dict:
    cleaned = clean_data(raw_data).result()
    transformed = transform_data(cleaned).result()
    return transformed

# Dùng kết quả của functional API trong graph
def orchestrator_node(state):
    processed_data = data_processor.invoke(state["raw_data"])
    return {"processed_data": processed_data}
```

## Chuyển đổi giữa các API

### Từ Functional sang Graph API

Khi workflow functional của bạn trở nên phức tạp, bạn có thể chuyển sang Graph API:

```python
# Trước: Functional API
@entrypoint(checkpointer=checkpointer)
def complex_workflow(input_data: dict) -> dict:
    step1 = process_step1(input_data).result()

    if step1["needs_analysis"]:
        analysis = analyze_data(step1).result()
        if analysis["confidence"] > 0.8:
            result = high_confidence_path(analysis).result()
        else:
            result = low_confidence_path(analysis).result()
    else:
        result = simple_path(step1).result()

    return result

# Sau: Graph API
class WorkflowState(TypedDict):
    input_data: dict
    step1_result: dict
    analysis: dict
    final_result: dict

def should_analyze(state):
    return "analyze" if state["step1_result"]["needs_analysis"] else "simple_path"

def confidence_check(state):
    return "high_confidence" if state["analysis"]["confidence"] > 0.8 else "low_confidence"

workflow = StateGraph(WorkflowState)
workflow.add_node("step1", process_step1_node)
workflow.add_conditional_edges("step1", should_analyze)
workflow.add_node("analyze", analyze_data_node)
workflow.add_conditional_edges("analyze", confidence_check)
# ... thêm các node và cạnh còn lại
```

### Từ Graph sang Functional API

Khi graph của bạn trở nên quá phức tạp cho các quy trình tuyến tính đơn giản:

```python
# Trước: Graph API bị lạm dụng quá mức
class SimpleState(TypedDict):
    input: str
    step1: str
    step2: str
    result: str

# Sau: Functional API đơn giản hoá
@entrypoint(checkpointer=checkpointer)
def simple_workflow(input_data: str) -> str:
    step1 = process_step1(input_data).result()
    step2 = process_step2(step1).result()
    return finalize_result(step2).result()
```

## Tóm tắt

Chọn **Graph API** khi bạn cần kiểm soát tường minh cấu trúc workflow, phân nhánh phức tạp, xử lý song song, hoặc lợi ích từ cộng tác nhóm.

Chọn **Functional API** khi bạn muốn thêm tính năng LangGraph vào code hiện có với thay đổi tối thiểu, có workflow tuyến tính đơn giản, hoặc cần khả năng prototype nhanh.

Cả hai API đều cung cấp cùng các tính năng lõi của LangGraph (persistence, streaming, human-in-the-loop, memory) nhưng đóng gói chúng theo các paradigm khác nhau để phù hợp với các phong cách phát triển và trường hợp sử dụng khác nhau.

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/choosing-apis.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
