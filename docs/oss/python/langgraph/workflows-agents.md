# Workflows và agent

Hướng dẫn này điểm qua các pattern workflow và agent phổ biến.

* Workflow có các đường code được xác định trước và được thiết kế để hoạt động theo một thứ tự nhất định.
* Agent thì linh hoạt (dynamic) và tự định nghĩa quy trình cũng như cách dùng tool của riêng mình.

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=c217c9ef517ee556cae3fc928a21dc55" alt="Agent Workflow" width="4572" height="2047" data-path="oss/images/agent_workflow.png" />

LangGraph mang lại nhiều lợi ích khi xây dựng agent và workflow, bao gồm [persistence](./persistence.md), [streaming](./streaming.md), và hỗ trợ debug cũng như [deployment](./deploy.md).

!!! tip ""
    Trace và so sánh các pattern workflow này với [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langgraph-workflows-agents). Làm theo [tracing quickstart](https://docs.langchain.com/langsmith/trace-with-langgraph) để xem dữ liệu chảy qua từng bước như thế nào. Chúng tôi cũng khuyến nghị bạn thiết lập [LangSmith Engine](https://docs.langchain.com/langsmith/engine) để theo dõi trace, phát hiện vấn đề, và đề xuất cách khắc phục.

## Cài đặt

Để xây dựng một workflow hoặc agent, bạn có thể dùng [bất kỳ chat model nào](https://docs.langchain.com/oss/python/integrations/chat) hỗ trợ structured output và tool calling. Ví dụ dưới đây dùng Anthropic:

1. Cài dependency:

```bash
pip install langchain_core langchain-anthropic langgraph
```

2. Khởi tạo LLM:

```python
import os
import getpass

from langchain_anthropic import ChatAnthropic

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("ANTHROPIC_API_KEY")

llm = ChatAnthropic(model="claude-sonnet-4-6")
```

## LLM và các bổ trợ (augmentation)

Workflow và hệ thống agentic đều dựa trên LLM và các bổ trợ (augmentation) khác nhau mà bạn thêm vào chúng. [Tool calling](../langchain/tools.md), [structured output](../langchain/structured-output.md), và [short term memory](../langchain/short-term-memory.md) là một vài lựa chọn để tuỳ chỉnh LLM theo nhu cầu của bạn.

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=7ea9656f46649b3ebac19e8309ae9006" alt="LLM augmentations" width="1152" height="778" data-path="oss/images/augmented_llm.png" />

```python
# Schema cho structured output
from pydantic import BaseModel, Field


class SearchQuery(BaseModel):
    search_query: str = Field(None, description="Query that is optimized web search.")
    justification: str = Field(
        None, description="Why this query is relevant to the user's request."
    )


# Bổ trợ LLM với schema cho structured output
structured_llm = llm.with_structured_output(SearchQuery)

# Gọi LLM đã được bổ trợ
output = structured_llm.invoke("How does Calcium CT score relate to high cholesterol?")

# Định nghĩa một tool
def multiply(a: int, b: int) -> int:
    return a * b

# Bổ trợ LLM với tool
llm_with_tools = llm.bind_tools([multiply])

# Gọi LLM với input kích hoạt tool call
msg = llm_with_tools.invoke("What is 2 times 3?")

# Lấy tool call
msg.tool_calls
```

## Prompt chaining

Prompt chaining là khi mỗi lần gọi LLM xử lý output của lần gọi trước đó. Thường dùng để thực hiện các tác vụ được định nghĩa rõ ràng có thể chia nhỏ thành các bước nhỏ hơn, có thể kiểm chứng được. Một số ví dụ gồm:

* Dịch tài liệu sang các ngôn ngữ khác nhau
* Xác minh tính nhất quán của nội dung được sinh ra

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=762dec147c31b8dc6ebb0857e236fc1f" alt="Prompt chaining" width="1412" height="444" data-path="oss/images/prompt_chain.png" />

=== "Graph API"
    ```python
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END
    from IPython.display import Image, display


    # State của graph
    class State(TypedDict):
        topic: str
        joke: str
        improved_joke: str
        final_joke: str


    # Các node
    def generate_joke(state: State):
        """Lần gọi LLM đầu tiên để sinh joke ban đầu"""

        msg = llm.invoke(f"Write a short joke about {state['topic']}")
        return {"joke": msg.content}


    def check_punchline(state: State):
        """Hàm gate kiểm tra xem joke có punchline hay không"""

        # Kiểm tra đơn giản - joke có chứa "?" hoặc "!" không
        if "?" in state["joke"] or "!" in state["joke"]:
            return "Pass"
        return "Fail"


    def improve_joke(state: State):
        """Lần gọi LLM thứ hai để cải thiện joke"""

        msg = llm.invoke(f"Make this joke funnier by adding wordplay: {state['joke']}")
        return {"improved_joke": msg.content}


    def polish_joke(state: State):
        """Lần gọi LLM thứ ba để đánh bóng lần cuối"""
        msg = llm.invoke(f"Add a surprising twist to this joke: {state['improved_joke']}")
        return {"final_joke": msg.content}


    # Xây dựng workflow
    workflow = StateGraph(State)

    # Thêm node
    workflow.add_node("generate_joke", generate_joke)
    workflow.add_node("improve_joke", improve_joke)
    workflow.add_node("polish_joke", polish_joke)

    # Thêm edge để kết nối các node
    workflow.add_edge(START, "generate_joke")
    workflow.add_conditional_edges(
        "generate_joke", check_punchline, {"Fail": "improve_joke", "Pass": END}
    )
    workflow.add_edge("improve_joke", "polish_joke")
    workflow.add_edge("polish_joke", END)

    # Compile
    chain = workflow.compile()

    # Hiển thị workflow
    display(Image(chain.get_graph().draw_mermaid_png()))

    # Gọi
    state = chain.invoke({"topic": "cats"})
    print("Initial joke:")
    print(state["joke"])
    print("\n--- --- ---\n")
    if "improved_joke" in state:
        print("Improved joke:")
        print(state["improved_joke"])
        print("\n--- --- ---\n")

        print("Final joke:")
        print(state["final_joke"])
    else:
        print("Final joke:")
        print(state["joke"])
    ```

=== "Functional API"
    ```python
    from langgraph.func import entrypoint, task


    # Task
    @task
    def generate_joke(topic: str):
        """Lần gọi LLM đầu tiên để sinh joke ban đầu"""
        msg = llm.invoke(f"Write a short joke about {topic}")
        return msg.content


    def check_punchline(joke: str):
        """Hàm gate kiểm tra xem joke có punchline hay không"""
        # Kiểm tra đơn giản - joke có chứa "?" hoặc "!" không
        if "?" in joke or "!" in joke:
            return "Fail"

        return "Pass"


    @task
    def improve_joke(joke: str):
        """Lần gọi LLM thứ hai để cải thiện joke"""
        msg = llm.invoke(f"Make this joke funnier by adding wordplay: {joke}")
        return msg.content


    @task
    def polish_joke(joke: str):
        """Lần gọi LLM thứ ba để đánh bóng lần cuối"""
        msg = llm.invoke(f"Add a surprising twist to this joke: {joke}")
        return msg.content


    @entrypoint()
    def prompt_chaining_workflow(topic: str):
        original_joke = generate_joke(topic).result()
        if check_punchline(original_joke) == "Pass":
            return original_joke

        improved_joke = improve_joke(original_joke).result()
        return polish_joke(improved_joke).result()

    # Gọi
    stream = prompt_chaining_workflow.stream_events("cats", version="v3")
    for snapshot in stream.values:
        print(snapshot)
        print("\n")
    ```

## Song song hoá (parallelization)

Với parallelization, các LLM làm việc đồng thời trên một tác vụ. Việc này được thực hiện bằng cách chạy nhiều tác vụ con độc lập cùng lúc, hoặc chạy cùng một tác vụ nhiều lần để kiểm tra các output khác nhau. Parallelization thường được dùng để:

* Chia nhỏ tác vụ con và chạy chúng song song, giúp tăng tốc độ
* Chạy tác vụ nhiều lần để kiểm tra các output khác nhau, giúp tăng độ tin cậy

Một số ví dụ gồm:

* Chạy một tác vụ con xử lý tài liệu để tìm từ khoá, và một tác vụ con khác để kiểm tra lỗi định dạng
* Chạy một tác vụ nhiều lần để chấm điểm độ chính xác của tài liệu dựa trên các tiêu chí khác nhau, như số lượng trích dẫn, số nguồn được dùng, và chất lượng của các nguồn

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=8afe3c427d8cede6fed1e4b2a5107b71" alt="parallelization.png" width="1020" height="684" data-path="oss/images/parallelization.png" />

=== "Graph API"
    ```python
    # State của graph
    class State(TypedDict):
        topic: str
        joke: str
        story: str
        poem: str
        combined_output: str


    # Các node
    def call_llm_1(state: State):
        """Lần gọi LLM đầu tiên để sinh joke ban đầu"""

        msg = llm.invoke(f"Write a joke about {state['topic']}")
        return {"joke": msg.content}


    def call_llm_2(state: State):
        """Lần gọi LLM thứ hai để sinh story"""

        msg = llm.invoke(f"Write a story about {state['topic']}")
        return {"story": msg.content}


    def call_llm_3(state: State):
        """Lần gọi LLM thứ ba để sinh poem"""

        msg = llm.invoke(f"Write a poem about {state['topic']}")
        return {"poem": msg.content}


    def aggregator(state: State):
        """Kết hợp joke, story và poem thành một output duy nhất"""

        combined = f"Here's a story, joke, and poem about {state['topic']}!\n\n"
        combined += f"STORY:\n{state['story']}\n\n"
        combined += f"JOKE:\n{state['joke']}\n\n"
        combined += f"POEM:\n{state['poem']}"
        return {"combined_output": combined}


    # Xây dựng workflow
    parallel_builder = StateGraph(State)

    # Thêm node
    parallel_builder.add_node("call_llm_1", call_llm_1)
    parallel_builder.add_node("call_llm_2", call_llm_2)
    parallel_builder.add_node("call_llm_3", call_llm_3)
    parallel_builder.add_node("aggregator", aggregator)

    # Thêm edge để kết nối các node
    parallel_builder.add_edge(START, "call_llm_1")
    parallel_builder.add_edge(START, "call_llm_2")
    parallel_builder.add_edge(START, "call_llm_3")
    parallel_builder.add_edge("call_llm_1", "aggregator")
    parallel_builder.add_edge("call_llm_2", "aggregator")
    parallel_builder.add_edge("call_llm_3", "aggregator")
    parallel_builder.add_edge("aggregator", END)
    parallel_workflow = parallel_builder.compile()

    # Hiển thị workflow
    display(Image(parallel_workflow.get_graph().draw_mermaid_png()))

    # Gọi
    state = parallel_workflow.invoke({"topic": "cats"})
    print(state["combined_output"])
    ```

=== "Functional API"
    ```python
    @task
    def call_llm_1(topic: str):
        """Lần gọi LLM đầu tiên để sinh joke ban đầu"""
        msg = llm.invoke(f"Write a joke about {topic}")
        return msg.content


    @task
    def call_llm_2(topic: str):
        """Lần gọi LLM thứ hai để sinh story"""
        msg = llm.invoke(f"Write a story about {topic}")
        return msg.content


    @task
    def call_llm_3(topic):
        """Lần gọi LLM thứ ba để sinh poem"""
        msg = llm.invoke(f"Write a poem about {topic}")
        return msg.content


    @task
    def aggregator(topic, joke, story, poem):
        """Kết hợp joke và story thành một output duy nhất"""

        combined = f"Here's a story, joke, and poem about {topic}!\n\n"
        combined += f"STORY:\n{story}\n\n"
        combined += f"JOKE:\n{joke}\n\n"
        combined += f"POEM:\n{poem}"
        return combined


    # Xây dựng workflow
    @entrypoint()
    def parallel_workflow(topic: str):
        joke_fut = call_llm_1(topic)
        story_fut = call_llm_2(topic)
        poem_fut = call_llm_3(topic)
        return aggregator(
            topic, joke_fut.result(), story_fut.result(), poem_fut.result()
        ).result()

    # Gọi
    stream = parallel_workflow.stream_events("cats", version="v3")
    for snapshot in stream.values:
        print(snapshot)
        print("\n")
    ```

## Định tuyến (routing)

Workflow định tuyến xử lý input rồi điều hướng chúng tới các tác vụ theo ngữ cảnh cụ thể. Điều này cho phép bạn định nghĩa các luồng chuyên biệt cho các tác vụ phức tạp. Ví dụ, một workflow được xây dựng để trả lời câu hỏi liên quan đến sản phẩm có thể xử lý loại câu hỏi trước, rồi định tuyến yêu cầu tới các quy trình cụ thể cho giá cả, hoàn tiền, trả hàng, v.v.

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=272e0e9b681b89cd7d35d5c812c50ee6" alt="routing.png" width="1214" height="678" data-path="oss/images/routing.png" />

=== "Graph API"
    ```python
    from typing_extensions import Literal
    from langchain.messages import HumanMessage, SystemMessage


    # Schema cho structured output dùng làm logic định tuyến
    class Route(BaseModel):
        step: Literal["poem", "story", "joke"] = Field(
            None, description="The next step in the routing process"
        )


    # Bổ trợ LLM với schema cho structured output
    router = llm.with_structured_output(Route)


    # State
    class State(TypedDict):
        input: str
        decision: str
        output: str


    # Các node
    def llm_call_1(state: State):
        """Viết một story"""

        result = llm.invoke(state["input"])
        return {"output": result.content}


    def llm_call_2(state: State):
        """Viết một joke"""

        result = llm.invoke(state["input"])
        return {"output": result.content}


    def llm_call_3(state: State):
        """Viết một poem"""

        result = llm.invoke(state["input"])
        return {"output": result.content}


    def llm_call_router(state: State):
        """Định tuyến input tới node phù hợp"""

        # Chạy LLM đã bổ trợ với structured output để làm logic định tuyến
        decision = router.invoke(
            [
                SystemMessage(
                    content="Route the input to story, joke, or poem based on the user's request."
                ),
                HumanMessage(content=state["input"]),
            ]
        )

        return {"decision": decision.step}


    # Hàm conditional edge để định tuyến tới node phù hợp
    def route_decision(state: State):
        # Trả về tên node bạn muốn ghé thăm tiếp theo
        if state["decision"] == "story":
            return "llm_call_1"
        elif state["decision"] == "joke":
            return "llm_call_2"
        elif state["decision"] == "poem":
            return "llm_call_3"


    # Xây dựng workflow
    router_builder = StateGraph(State)

    # Thêm node
    router_builder.add_node("llm_call_1", llm_call_1)
    router_builder.add_node("llm_call_2", llm_call_2)
    router_builder.add_node("llm_call_3", llm_call_3)
    router_builder.add_node("llm_call_router", llm_call_router)

    # Thêm edge để kết nối các node
    router_builder.add_edge(START, "llm_call_router")
    router_builder.add_conditional_edges(
        "llm_call_router",
        route_decision,
        {  # Tên trả về bởi route_decision : Tên node tiếp theo cần ghé thăm
            "llm_call_1": "llm_call_1",
            "llm_call_2": "llm_call_2",
            "llm_call_3": "llm_call_3",
        },
    )
    router_builder.add_edge("llm_call_1", END)
    router_builder.add_edge("llm_call_2", END)
    router_builder.add_edge("llm_call_3", END)

    # Compile workflow
    router_workflow = router_builder.compile()

    # Hiển thị workflow
    display(Image(router_workflow.get_graph().draw_mermaid_png()))

    # Gọi
    state = router_workflow.invoke({"input": "Write me a joke about cats"})
    print(state["output"])
    ```

=== "Functional API"
    ```python
    from typing_extensions import Literal
    from pydantic import BaseModel
    from langchain.messages import HumanMessage, SystemMessage


    # Schema cho structured output dùng làm logic định tuyến
    class Route(BaseModel):
        step: Literal["poem", "story", "joke"] = Field(
            None, description="The next step in the routing process"
        )


    # Bổ trợ LLM với schema cho structured output
    router = llm.with_structured_output(Route)


    @task
    def llm_call_1(input_: str):
        """Viết một story"""
        result = llm.invoke(input_)
        return result.content


    @task
    def llm_call_2(input_: str):
        """Viết một joke"""
        result = llm.invoke(input_)
        return result.content


    @task
    def llm_call_3(input_: str):
        """Viết một poem"""
        result = llm.invoke(input_)
        return result.content


    def llm_call_router(input_: str):
        """Định tuyến input tới node phù hợp"""
        # Chạy LLM đã bổ trợ với structured output để làm logic định tuyến
        decision = router.invoke(
            [
                SystemMessage(
                    content="Route the input to story, joke, or poem based on the user's request."
                ),
                HumanMessage(content=input_),
            ]
        )
        return decision.step


    # Tạo workflow
    @entrypoint()
    def router_workflow(input_: str):
        next_step = llm_call_router(input_)
        if next_step == "story":
            llm_call = llm_call_1
        elif next_step == "joke":
            llm_call = llm_call_2
        elif next_step == "poem":
            llm_call = llm_call_3

        return llm_call(input_).result()

    # Gọi
    stream = router_workflow.stream_events("Write me a joke about cats", version="v3")
    for snapshot in stream.values:
        print(snapshot)
        print("\n")
    ```

## Orchestrator-worker

Trong cấu hình orchestrator-worker, orchestrator sẽ:

* Chia nhỏ tác vụ thành các tác vụ con
* Giao các tác vụ con cho worker
* Tổng hợp output của các worker thành một kết quả cuối cùng

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=2e423c67cd4f12e049cea9c169ff0676" alt="worker.png" width="1486" height="548" data-path="oss/images/worker.png" />

Workflow orchestrator-worker mang lại tính linh hoạt cao hơn và thường được dùng khi các tác vụ con không thể định nghĩa trước theo cách [parallelization](#song-song-hoa-parallelization) cho phép. Điều này phổ biến với các workflow viết code hoặc cần cập nhật nội dung trên nhiều file. Ví dụ, một workflow cần cập nhật hướng dẫn cài đặt cho nhiều thư viện Python trên một số lượng tài liệu không xác định trước có thể dùng pattern này.

=== "Graph API"
    ```python
    from typing import Annotated, List
    import operator


    # Schema cho structured output dùng trong lập kế hoạch
    class Section(BaseModel):
        name: str = Field(
            description="Name for this section of the report.",
        )
        description: str = Field(
            description="Brief overview of the main topics and concepts to be covered in this section.",
        )


    class Sections(BaseModel):
        sections: List[Section] = Field(
            description="Sections of the report.",
        )


    # Bổ trợ LLM với schema cho structured output
    planner = llm.with_structured_output(Sections)
    ```

=== "Functional API"
    ```python
    from typing import List


    # Schema cho structured output dùng trong lập kế hoạch
    class Section(BaseModel):
        name: str = Field(
            description="Name for this section of the report.",
        )
        description: str = Field(
            description="Brief overview of the main topics and concepts to be covered in this section.",
        )


    class Sections(BaseModel):
        sections: List[Section] = Field(
            description="Sections of the report.",
        )


    # Bổ trợ LLM với schema cho structured output
    planner = llm.with_structured_output(Sections)


    @task
    def orchestrator(topic: str):
        """Orchestrator sinh ra kế hoạch cho báo cáo"""
        # Sinh query
        report_sections = planner.invoke(
            [
                SystemMessage(content="Generate a plan for the report."),
                HumanMessage(content=f"Here is the report topic: {topic}"),
            ]
        )

        return report_sections.sections


    @task
    def llm_call(section: Section):
        """Worker viết một phần của báo cáo"""

        # Sinh section
        result = llm.invoke(
            [
                SystemMessage(content="Write a report section."),
                HumanMessage(
                    content=f"Here is the section name: {section.name} and description: {section.description}"
                ),
            ]
        )

        # Ghi section đã cập nhật vào các section đã hoàn thành
        return result.content


    @task
    def synthesizer(completed_sections: list[str]):
        """Tổng hợp báo cáo hoàn chỉnh từ các section"""
        final_report = "\n\n---\n\n".join(completed_sections)
        return final_report


    @entrypoint()
    def orchestrator_worker(topic: str):
        sections = orchestrator(topic).result()
        section_futures = [llm_call(section) for section in sections]
        final_report = synthesizer(
            [section_fut.result() for section_fut in section_futures]
        ).result()
        return final_report

    # Gọi
    report = orchestrator_worker.invoke("Create a report on LLM scaling laws")
    from IPython.display import Markdown
    Markdown(report)
    ```

### Tạo worker trong LangGraph

Workflow orchestrator-worker rất phổ biến và LangGraph có hỗ trợ dựng sẵn cho chúng. API `Send` cho phép bạn tạo động các node worker và gửi cho chúng input cụ thể. Mỗi worker có state riêng, và tất cả output của worker được ghi vào một state key dùng chung mà graph orchestrator có thể truy cập. Điều này cho phép orchestrator truy cập toàn bộ output của worker và tổng hợp chúng thành một output cuối cùng. Ví dụ dưới đây lặp qua một danh sách section và dùng API `Send` để gửi một section cho mỗi worker.

```python
from langgraph.types import Send


# State của graph
class State(TypedDict):
    topic: str  # Chủ đề báo cáo
    sections: list[Section]  # Danh sách các section của báo cáo
    completed_sections: Annotated[
        list, operator.add
    ]  # Tất cả worker ghi vào key này song song
    final_report: str  # Báo cáo cuối cùng


# State của worker
class WorkerState(TypedDict):
    section: Section
    completed_sections: Annotated[list, operator.add]


# Các node
def orchestrator(state: State):
    """Orchestrator sinh ra kế hoạch cho báo cáo"""

    # Sinh query
    report_sections = planner.invoke(
        [
            SystemMessage(content="Generate a plan for the report."),
            HumanMessage(content=f"Here is the report topic: {state['topic']}"),
        ]
    )

    return {"sections": report_sections.sections}


def llm_call(state: WorkerState):
    """Worker viết một phần của báo cáo"""

    # Sinh section
    section = llm.invoke(
        [
            SystemMessage(
                content="Write a report section following the provided name and description. Include no preamble for each section. Use markdown formatting."
            ),
            HumanMessage(
                content=f"Here is the section name: {state['section'].name} and description: {state['section'].description}"
            ),
        ]
    )

    # Ghi section đã cập nhật vào các section đã hoàn thành
    return {"completed_sections": [section.content]}


def synthesizer(state: State):
    """Tổng hợp báo cáo hoàn chỉnh từ các section"""

    # Danh sách các section đã hoàn thành
    completed_sections = state["completed_sections"]

    # Định dạng section đã hoàn thành thành str để dùng làm ngữ cảnh cho section cuối
    completed_report_sections = "\n\n---\n\n".join(completed_sections)

    return {"final_report": completed_report_sections}


# Hàm conditional edge để tạo các worker llm_call, mỗi worker viết một section của báo cáo
def assign_workers(state: State):
    """Gán một worker cho mỗi section trong kế hoạch"""

    # Khởi chạy việc viết section song song qua API Send()
    return [Send("llm_call", {"section": s}) for s in state["sections"]]


# Xây dựng workflow
orchestrator_worker_builder = StateGraph(State)

# Thêm các node
orchestrator_worker_builder.add_node("orchestrator", orchestrator)
orchestrator_worker_builder.add_node("llm_call", llm_call)
orchestrator_worker_builder.add_node("synthesizer", synthesizer)

# Thêm edge để kết nối các node
orchestrator_worker_builder.add_edge(START, "orchestrator")
orchestrator_worker_builder.add_conditional_edges(
    "orchestrator", assign_workers, ["llm_call"]
)
orchestrator_worker_builder.add_edge("llm_call", "synthesizer")
orchestrator_worker_builder.add_edge("synthesizer", END)

# Compile workflow
orchestrator_worker = orchestrator_worker_builder.compile()

# Hiển thị workflow
display(Image(orchestrator_worker.get_graph().draw_mermaid_png()))

# Gọi
state = orchestrator_worker.invoke({"topic": "Create a report on LLM scaling laws"})

from IPython.display import Markdown
Markdown(state["final_report"])
```

## Evaluator-optimizer

Trong workflow evaluator-optimizer, một lần gọi LLM tạo ra response và lần gọi khác đánh giá response đó. Nếu evaluator hoặc một [human-in-the-loop](./interrupts.md) xác định response cần cải thiện, feedback sẽ được cung cấp và response được tạo lại. Vòng lặp này tiếp tục cho tới khi một response chấp nhận được được sinh ra.

Workflow evaluator-optimizer thường được dùng khi có tiêu chí thành công cụ thể cho một tác vụ, nhưng cần lặp lại để đáp ứng tiêu chí đó. Ví dụ, không phải lúc nào cũng có sự khớp hoàn hảo khi dịch văn bản giữa hai ngôn ngữ. Có thể cần vài lần lặp để tạo ra một bản dịch có cùng ý nghĩa giữa hai ngôn ngữ.

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=9bd0474f42b6040b14ed6968a9ab4e3c" alt="evaluator_optimizer.png" width="1004" height="340" data-path="oss/images/evaluator_optimizer.png" />

=== "Graph API"
    ```python
    # State của graph
    class State(TypedDict):
        joke: str
        topic: str
        feedback: str
        funny_or_not: str


    # Schema cho structured output dùng trong đánh giá
    class Feedback(BaseModel):
        grade: Literal["funny", "not funny"] = Field(
            description="Decide if the joke is funny or not.",
        )
        feedback: str = Field(
            description="If the joke is not funny, provide feedback on how to improve it.",
        )


    # Bổ trợ LLM với schema cho structured output
    evaluator = llm.with_structured_output(Feedback)


    # Các node
    def llm_call_generator(state: State):
        """LLM sinh ra một joke"""

        if state.get("feedback"):
            msg = llm.invoke(
                f"Write a joke about {state['topic']} but take into account the feedback: {state['feedback']}"
            )
        else:
            msg = llm.invoke(f"Write a joke about {state['topic']}")
        return {"joke": msg.content}


    def llm_call_evaluator(state: State):
        """LLM đánh giá joke"""

        grade = evaluator.invoke(f"Grade the joke {state['joke']}")
        return {"funny_or_not": grade.grade, "feedback": grade.feedback}


    # Hàm conditional edge định tuyến ngược lại tới node sinh joke hoặc kết thúc dựa trên feedback của evaluator
    def route_joke(state: State):
        """Định tuyến ngược lại tới node sinh joke hoặc kết thúc dựa trên feedback của evaluator"""

        if state["funny_or_not"] == "funny":
            return "Accepted"
        elif state["funny_or_not"] == "not funny":
            return "Rejected + Feedback"


    # Xây dựng workflow
    optimizer_builder = StateGraph(State)

    # Thêm các node
    optimizer_builder.add_node("llm_call_generator", llm_call_generator)
    optimizer_builder.add_node("llm_call_evaluator", llm_call_evaluator)

    # Thêm edge để kết nối các node
    optimizer_builder.add_edge(START, "llm_call_generator")
    optimizer_builder.add_edge("llm_call_generator", "llm_call_evaluator")
    optimizer_builder.add_conditional_edges(
        "llm_call_evaluator",
        route_joke,
        {  # Tên trả về bởi route_joke : Tên node tiếp theo cần ghé thăm
            "Accepted": END,
            "Rejected + Feedback": "llm_call_generator",
        },
    )

    # Compile workflow
    optimizer_workflow = optimizer_builder.compile()

    # Hiển thị workflow
    display(Image(optimizer_workflow.get_graph().draw_mermaid_png()))

    # Gọi
    state = optimizer_workflow.invoke({"topic": "Cats"})
    print(state["joke"])
    ```

=== "Functional API"
    ```python
    # Schema cho structured output dùng trong đánh giá
    class Feedback(BaseModel):
        grade: Literal["funny", "not funny"] = Field(
            description="Decide if the joke is funny or not.",
        )
        feedback: str = Field(
            description="If the joke is not funny, provide feedback on how to improve it.",
        )


    # Bổ trợ LLM với schema cho structured output
    evaluator = llm.with_structured_output(Feedback)


    # Các node
    @task
    def llm_call_generator(topic: str, feedback: Feedback):
        """LLM sinh ra một joke"""
        if feedback:
            msg = llm.invoke(
                f"Write a joke about {topic} but take into account the feedback: {feedback}"
            )
        else:
            msg = llm.invoke(f"Write a joke about {topic}")
        return msg.content


    @task
    def llm_call_evaluator(joke: str):
        """LLM đánh giá joke"""
        feedback = evaluator.invoke(f"Grade the joke {joke}")
        return feedback


    @entrypoint()
    def optimizer_workflow(topic: str):
        feedback = None
        while True:
            joke = llm_call_generator(topic, feedback).result()
            feedback = llm_call_evaluator(joke).result()
            if feedback.grade == "funny":
                break

        return joke

    # Gọi
    stream = optimizer_workflow.stream_events("Cats", version="v3")
    for snapshot in stream.values:
        print(snapshot)
        print("\n")
    ```

## Agent

Agent thường được triển khai dưới dạng một LLM thực hiện các hành động bằng cách dùng [tool](../langchain/tools.md). Chúng hoạt động trong các vòng lặp phản hồi liên tục, và được dùng trong các tình huống mà vấn đề và giải pháp không thể đoán trước. Agent có nhiều quyền tự chủ hơn workflow, và có thể tự quyết định về các tool sẽ dùng và cách giải quyết vấn đề. Bạn vẫn có thể định nghĩa bộ tool khả dụng và các quy tắc hướng dẫn cách agent hành xử.

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=bd8da41dbf8b5e6fc9ea6bb10cb63e38" alt="agent.png" width="1732" height="712" data-path="oss/images/agent.png" />

!!! note ""
    Để bắt đầu với agent, xem [quickstart](../langchain/quickstart.md) hoặc đọc thêm về [cách chúng hoạt động](../langchain/agents.md) trong LangChain.

```python
from langchain.tools import tool


# Định nghĩa tool
@tool
def multiply(a: int, b: int) -> int:
    """Multiply `a` and `b`.

    Args:
        a: First int
        b: Second int
    """
    return a * b


@tool
def add(a: int, b: int) -> int:
    """Adds `a` and `b`.

    Args:
        a: First int
        b: Second int
    """
    return a + b


@tool
def divide(a: int, b: int) -> float:
    """Divide `a` and `b`.

    Args:
        a: First int
        b: Second int
    """
    return a / b


# Bổ trợ LLM với tool
tools = [add, multiply, divide]
tools_by_name = {tool.name: tool for tool in tools}
llm_with_tools = llm.bind_tools(tools)
```

=== "Graph API"
    ```python
    from langgraph.graph import MessagesState
    from langchain.messages import SystemMessage, HumanMessage, ToolMessage


    # Các node
    def llm_call(state: MessagesState):
        """LLM quyết định có gọi tool hay không"""

        return {
            "messages": [
                llm_with_tools.invoke(
                    [
                        SystemMessage(
                            content="You are a helpful assistant tasked with performing arithmetic on a set of inputs."
                        )
                    ]
                    + state["messages"]
                )
            ]
        }


    def tool_node(state: MessagesState):
        """Thực hiện tool call"""

        result = []
        for tool_call in state["messages"][-1].tool_calls:
            tool = tools_by_name[tool_call["name"]]
            observation = tool.invoke(tool_call["args"])
            result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
        return {"messages": result}


    # Hàm conditional edge định tuyến tới node tool hoặc kết thúc dựa trên việc LLM có tool call hay không
    def should_continue(state: MessagesState) -> Literal["tool_node", END]:
        """Quyết định có tiếp tục vòng lặp hay dừng dựa trên việc LLM có tool call hay không"""

        messages = state["messages"]
        last_message = messages[-1]

        # Nếu LLM có tool call, thực hiện một hành động
        if last_message.tool_calls:
            return "tool_node"

        # Nếu không, dừng lại (trả lời người dùng)
        return END


    # Xây dựng workflow
    agent_builder = StateGraph(MessagesState)

    # Thêm node
    agent_builder.add_node("llm_call", llm_call)
    agent_builder.add_node("tool_node", tool_node)

    # Thêm edge để kết nối các node
    agent_builder.add_edge(START, "llm_call")
    agent_builder.add_conditional_edges(
        "llm_call",
        should_continue,
        ["tool_node", END]
    )
    agent_builder.add_edge("tool_node", "llm_call")

    # Compile agent
    agent = agent_builder.compile()

    # Hiển thị agent
    display(Image(agent.get_graph(xray=True).draw_mermaid_png()))

    # Gọi
    messages = [HumanMessage(content="Add 3 and 4.")]
    messages = agent.invoke({"messages": messages})
    for m in messages["messages"]:
        m.pretty_print()
    ```

=== "Functional API"
    ```python
    from langgraph.graph import add_messages
    from langchain.messages import (
        SystemMessage,
        HumanMessage,
        ToolCall,
    )
    from langchain_core.messages import BaseMessage


    @task
    def call_llm(messages: list[BaseMessage]):
        """LLM quyết định có gọi tool hay không"""
        return llm_with_tools.invoke(
            [
                SystemMessage(
                    content="You are a helpful assistant tasked with performing arithmetic on a set of inputs."
                )
            ]
            + messages
        )


    @task
    def call_tool(tool_call: ToolCall):
        """Thực hiện tool call"""
        tool = tools_by_name[tool_call["name"]]
        return tool.invoke(tool_call)


    @entrypoint()
    def agent(messages: list[BaseMessage]):
        llm_response = call_llm(messages).result()

        while True:
            if not llm_response.tool_calls:
                break

            # Thực thi tool
            tool_result_futures = [
                call_tool(tool_call) for tool_call in llm_response.tool_calls
            ]
            tool_results = [fut.result() for fut in tool_result_futures]
            messages = add_messages(messages, [llm_response, *tool_results])
            llm_response = call_llm(messages).result()

        messages = add_messages(messages, llm_response)
        return messages

    # Gọi
    messages = [HumanMessage(content="Add 3 and 4.")]
    stream = agent.stream_events(messages, version="v3")
    for snapshot in stream.values:
        print(snapshot)
        print("\n")
    ```

### ToolNode

[`ToolNode`](https://reference.langchain.com/python/langgraph/agents/#langgraph.prebuilt.tool_node.ToolNode) là một node dựng sẵn thực thi tool trong workflow LangGraph. Nó tự động xử lý việc thực thi tool song song, xử lý lỗi, và inject state.

Dùng [`ToolNode`](https://reference.langchain.com/python/langgraph/agents/#langgraph.prebuilt.tool_node.ToolNode) khi bạn cần kiểm soát chi tiết cách graph thực thi tool. Đây là khối xây dựng cốt lõi cho việc thực thi tool trong nhiều pattern agent LangGraph.

```python
from langchain.tools import tool
from langgraph.prebuilt import ToolNode
from langgraph.graph import MessagesState, StateGraph

@tool
def search(query: str) -> str:
    """Search for information."""
    return f"Results for: {query}"

@tool
def calculator(expression: str) -> str:
    """Evaluate a math expression."""
    return str(eval(expression))

builder = StateGraph(MessagesState)
builder.add_node("tools", ToolNode([search, calculator]))
# ... thêm các node và edge khác
graph = builder.compile()
```

#### Truy cập state và context của graph từ tool

Tool được thực thi bởi `ToolNode` nhận các argument do model sinh ra làm
argument đầu tiên. Để đọc dữ liệu phía graph mà không do model sinh ra, dùng
một trong các lựa chọn sau:

* Trong Python, đọc state và context giới hạn theo run từ argument
  [`ToolRuntime`](https://reference.langchain.com/python/langchain/tools/#langchain.tools.ToolRuntime) được inject.
* Trong JavaScript, đọc state và context giới hạn theo run từ argument thứ hai
  của tool, có kiểu [`ToolRuntime`](https://reference.langchain.com/python/langchain/tools/#langchain.tools.ToolRuntime).

!!! note ""
    Tool chỉ có thể truy cập các giá trị state được truyền vào `ToolNode`. Khi
    `ToolNode` được thêm trực tiếp làm một node `StateGraph`, input đó là state
    hiện tại của graph. Nếu bạn tự gọi một `ToolNode` từ một node khác, hãy truyền
    toàn bộ state khi tool cần các field state tuỳ chỉnh. Ví dụ, `tool_node.invoke(state)`
    hoặc `toolNode.invoke(state, config)` sẽ expose toàn bộ state, trong khi chỉ truyền
    `{"messages": state["messages"]}` hoặc `{ messages: state.messages }` chỉ expose
    `messages`.

```python
from dataclasses import dataclass

from langchain.messages import AIMessage
from langchain.tools import ToolRuntime, tool
from langgraph.graph import MessagesState, START, StateGraph
from langgraph.prebuilt import ToolNode


class State(MessagesState):
    user_id: str


@dataclass
class Context:
    organization_id: str


@tool
def get_user_info(runtime: ToolRuntime[Context, State]) -> str:
    """Look up user information."""
    # Đọc state hiện tại của graph được truyền vào ToolNode.
    user_id = runtime.state["user_id"]

    # Đọc các giá trị tường minh theo từng run không thuộc state của graph.
    organization_id = runtime.context.organization_id

    return f"User {user_id} in organization {organization_id}"


builder = StateGraph(State, context_schema=Context)
builder.add_node("tools", ToolNode([get_user_info]))
builder.add_edge(START, "tools")
graph = builder.compile()

result = graph.invoke(
    {
        "messages": [
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "name": "get_user_info",
                        "args": {},
                        "id": "call_user_info",
                    }
                ],
            )
        ],
        "user_id": "user_123",
    },
    context=Context(organization_id="org_456"),
)
```

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/workflows-agents.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
