# Agent Evals

> Đánh giá trajectory của agent bằng so khớp deterministic hoặc evaluator LLM-as-judge với AgentEvals và LangSmith.

Evaluation ("evals") đo lường agent của bạn hoạt động tốt đến đâu bằng cách đánh giá execution trajectory (đường đi thực thi), tức chuỗi message và tool call mà nó tạo ra. Khác với [integration test](integration-testing.md) chỉ xác minh tính đúng đắn cơ bản, evals chấm điểm hành vi của agent dựa trên một tham chiếu hoặc rubric, giúp phát hiện regression khi bạn thay đổi prompt, tool, hoặc model.

Một evaluator là một hàm nhận đầu ra của agent (và tuỳ chọn cả reference output) rồi trả về điểm số:

```python
def evaluator(*, outputs: dict, reference_outputs: dict):
    output_messages = outputs["messages"]
    reference_messages = reference_outputs["messages"]
    score = compare_messages(output_messages, reference_messages)
    return {"key": "evaluator_score", "score": score}
```

Package [`agentevals`](https://github.com/langchain-ai/agentevals) cung cấp sẵn các evaluator cho trajectory của agent. Bạn có thể đánh giá bằng cách thực hiện **trajectory match** (so sánh deterministic) hoặc dùng **LLM judge** (đánh giá định tính):

| Cách tiếp cận | Khi nào dùng |
| --- | --- |
| [Trajectory match](#trajectory-match-evaluator) | Bạn biết trước tool call kỳ vọng và muốn kiểm tra nhanh, deterministic, không tốn chi phí |
| [LLM-as-judge](#llm-as-judge-evaluator) | Bạn muốn đánh giá chất lượng tổng thể và khả năng suy luận mà không có kỳ vọng chặt chẽ |

## Cài đặt AgentEvals

=== "pip"

    ```bash
    pip install -U agentevals
    ```

=== "uv"

    ```bash
    uv add agentevals
    ```

Hoặc clone trực tiếp [repository AgentEvals](https://github.com/langchain-ai/agentevals).

## Trajectory match evaluator

AgentEvals cung cấp hàm `create_trajectory_match_evaluator` để so khớp trajectory của agent với một tham chiếu. Có bốn chế độ:

| Chế độ | Mô tả | Use case |
| --- | --- | --- |
| `strict` | Khớp chính xác cấu trúc message và tool call theo cùng thứ tự (nội dung message có thể khác) | Kiểm thử chuỗi cụ thể (ví dụ: tra cứu policy trước khi authorize) |
| `unordered` | Cùng cấu trúc message và tool call như tham chiếu, nhưng tool call có thể xảy ra theo bất kỳ thứ tự nào | Xác minh việc truy xuất thông tin khi thứ tự không quan trọng |
| `subset` | Agent chỉ gọi các tool có trong tham chiếu (không thêm) | Đảm bảo agent không vượt quá phạm vi kỳ vọng |
| `superset` | Agent gọi ít nhất các tool trong tham chiếu (cho phép thêm) | Xác minh các hành động tối thiểu bắt buộc đã được thực hiện |

Các ví dụ dưới đây dùng chung một thiết lập: một agent có tool `get_weather`:

```python
from langchain.agents import create_agent
from langchain.tools import tool
from langchain.messages import HumanMessage, AIMessage, ToolMessage
from agentevals.trajectory.match import create_trajectory_match_evaluator


@tool
def get_weather(city: str):
    """Get weather information for a city."""
    return f"It's 75 degrees and sunny in {city}."

agent = create_agent("claude-sonnet-4-6", tools=[get_weather])
```

**Strict match**

Chế độ `strict` đảm bảo trajectory chứa các message giống hệt nhau theo cùng thứ tự với cùng tool call, dù cho phép khác biệt về nội dung message. Điều này hữu ích khi bạn cần bắt buộc một chuỗi thao tác cụ thể, ví dụ yêu cầu tra cứu policy trước khi authorize một hành động.

```python
evaluator = create_trajectory_match_evaluator(  # [!code highlight]
    trajectory_match_mode="strict",  # [!code highlight]
)  # [!code highlight]

def test_weather_tool_called_strict():
    result = agent.invoke({
        "messages": [HumanMessage(content="What's the weather in San Francisco?")]
    })

    reference_trajectory = [
        HumanMessage(content="What's the weather in San Francisco?"),
        AIMessage(content="", tool_calls=[
            {"id": "call_1", "name": "get_weather", "args": {"city": "San Francisco"}}
        ]),
        ToolMessage(content="It's 75 degrees and sunny in San Francisco.", tool_call_id="call_1"),
        AIMessage(content="The weather in San Francisco is 75 degrees and sunny."),
    ]

    evaluation = evaluator(
        outputs=result["messages"],
        reference_outputs=reference_trajectory
    )
    # {
    #     'key': 'trajectory_strict_match',
    #     'score': True,
    #     'comment': None,
    # }
    assert evaluation["score"] is True
```

**Unordered match**

Chế độ `unordered` cho phép cùng các tool call theo bất kỳ thứ tự nào. Điều này hữu ích khi bạn muốn xác minh thông tin cụ thể đã được truy xuất nhưng không quan tâm thứ tự. Ví dụ, một agent kiểm tra cả thời tiết lẫn sự kiện của một thành phố bằng các tool call khác nhau.

```python
@tool
def get_events(city: str):
    """Get events happening in a city."""
    return f"Concert at the park in {city} tonight."

agent = create_agent("claude-sonnet-4-6", tools=[get_weather, get_events])

evaluator = create_trajectory_match_evaluator(  # [!code highlight]
    trajectory_match_mode="unordered",  # [!code highlight]
)  # [!code highlight]

def test_multiple_tools_any_order():
    result = agent.invoke({
        "messages": [HumanMessage(content="What's happening in SF today?")]
    })

    reference_trajectory = [
        HumanMessage(content="What's happening in SF today?"),
        AIMessage(content="", tool_calls=[
            {"id": "call_1", "name": "get_events", "args": {"city": "SF"}},
            {"id": "call_2", "name": "get_weather", "args": {"city": "SF"}},
        ]),
        ToolMessage(content="Concert at the park in SF tonight.", tool_call_id="call_1"),
        ToolMessage(content="It's 75 degrees and sunny in SF.", tool_call_id="call_2"),
        AIMessage(content="Today in SF: 75 degrees and sunny with a concert at the park tonight."),
    ]

    evaluation = evaluator(
        outputs=result["messages"],
        reference_outputs=reference_trajectory,
    )
    assert evaluation["score"] is True
```

**Subset và superset match**

Chế độ `superset` và `subset` so khớp trajectory một phần. Chế độ `superset` xác minh agent đã gọi ít nhất các tool trong reference trajectory, cho phép gọi thêm tool khác. Chế độ `subset` đảm bảo agent không gọi tool nào ngoài các tool có trong tham chiếu.

```python
@tool
def get_detailed_forecast(city: str):
    """Get detailed weather forecast for a city."""
    return f"Detailed forecast for {city}: sunny all week."

agent = create_agent("claude-sonnet-4-6", tools=[get_weather, get_detailed_forecast])

evaluator = create_trajectory_match_evaluator(  # [!code highlight]
    trajectory_match_mode="superset",  # [!code highlight]
)  # [!code highlight]

def test_agent_calls_required_tools_plus_extra():
    result = agent.invoke({
        "messages": [HumanMessage(content="What's the weather in Boston?")]
    })

    # Reference chỉ yêu cầu get_weather, nhưng agent có thể gọi thêm tool khác
    reference_trajectory = [
        HumanMessage(content="What's the weather in Boston?"),
        AIMessage(content="", tool_calls=[
            {"id": "call_1", "name": "get_weather", "args": {"city": "Boston"}},
        ]),
        ToolMessage(content="It's 75 degrees and sunny in Boston.", tool_call_id="call_1"),
        AIMessage(content="The weather in Boston is 75 degrees and sunny."),
    ]

    evaluation = evaluator(
        outputs=result["messages"],
        reference_outputs=reference_trajectory,
    )
    assert evaluation["score"] is True
```

!!! info "Thông tin"
    Bạn cũng có thể đặt thuộc tính `tool_args_match_mode` và/hoặc `tool_args_match_overrides` để tuỳ chỉnh cách evaluator xem xét việc hai tool call bằng nhau giữa trajectory thực tế và tham chiếu. Mặc định, chỉ tool call cùng tên tool với cùng argument mới được coi là bằng nhau. Xem [repository](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#tool-args-match-modes) để biết thêm chi tiết.

## LLM-as-judge evaluator

Bạn có thể dùng một LLM để đánh giá đường đi thực thi của agent bằng hàm `create_trajectory_llm_as_judge`. Khác với trajectory match evaluator, hàm này không cần reference trajectory, nhưng bạn vẫn có thể cung cấp nếu có.

**Không có reference trajectory**

```python
from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT

evaluator = create_trajectory_llm_as_judge(  # [!code highlight]
    model="openai:o3-mini",  # [!code highlight]
    prompt=TRAJECTORY_ACCURACY_PROMPT,  # [!code highlight]
)  # [!code highlight]

def test_trajectory_quality():
    result = agent.invoke({
        "messages": [HumanMessage(content="What's the weather in Seattle?")]
    })

    evaluation = evaluator(
        outputs=result["messages"],
    )
    assert evaluation["score"] is True
```

**Có reference trajectory**

Nếu bạn có reference trajectory, dùng prompt dựng sẵn `TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE`:

```python
from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE

evaluator = create_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE,
)
evaluation = evaluator(
    outputs=result["messages"],
    reference_outputs=reference_trajectory,
)
```

!!! info "Thông tin"
    Để cấu hình chi tiết hơn về cách LLM đánh giá trajectory, xem [repository](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#trajectory-llm-as-judge).

### Hỗ trợ Async

Mọi evaluator của `agentevals` đều hỗ trợ Python asyncio. Phiên bản async có sẵn bằng cách thêm `async` sau `create_` trong tên hàm.

**Ví dụ judge và evaluator dạng async**

```python
from agentevals.trajectory.llm import create_async_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT
from agentevals.trajectory.match import create_async_trajectory_match_evaluator

async_judge = create_async_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT,
)

async_evaluator = create_async_trajectory_match_evaluator(
    trajectory_match_mode="strict",
)

async def test_async_evaluation():
    result = await agent.ainvoke({
        "messages": [HumanMessage(content="What's the weather?")]
    })

    evaluation = await async_judge(outputs=result["messages"])
    assert evaluation["score"] is True
```

## Chạy evals trong LangSmith

Để theo dõi experiment theo thời gian, ghi log kết quả evaluator vào [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-test-evals). Trước tiên, thiết lập các biến môi trường cần thiết:

```bash
export LANGSMITH_API_KEY="your_langsmith_api_key"
export LANGSMITH_TRACING="true"
```

LangSmith cung cấp hai cách tiếp cận chính để chạy evaluation: tích hợp [pytest](https://docs.langchain.com/langsmith/pytest) và hàm `evaluate`.

**Dùng tích hợp pytest**

```python
import pytest
from langsmith import testing as t
from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT

trajectory_evaluator = create_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT,
)

@pytest.mark.langsmith
def test_trajectory_accuracy():
    result = agent.invoke({
        "messages": [HumanMessage(content="What's the weather in SF?")]
    })

    reference_trajectory = [
        HumanMessage(content="What's the weather in SF?"),
        AIMessage(content="", tool_calls=[
            {"id": "call_1", "name": "get_weather", "args": {"city": "SF"}},
        ]),
        ToolMessage(content="It's 75 degrees and sunny in SF.", tool_call_id="call_1"),
        AIMessage(content="The weather in SF is 75 degrees and sunny."),
    ]

    t.log_inputs({})
    t.log_outputs({"messages": result["messages"]})
    t.log_reference_outputs({"messages": reference_trajectory})

    trajectory_evaluator(
        outputs=result["messages"],
        reference_outputs=reference_trajectory
    )
```

Chạy evaluation bằng pytest:

```bash
pytest test_trajectory.py --langsmith-output
```

**Dùng hàm evaluate**

Tạo một [LangSmith dataset](https://docs.langchain.com/langsmith/manage-datasets) và dùng hàm `evaluate`. Dataset phải có schema sau:

* **input**: `{"messages": [...]}` message đầu vào để gọi agent.
* **output**: `{"messages": [...]}` lịch sử message kỳ vọng trong output của agent. Với trajectory evaluation, bạn có thể chọn chỉ giữ lại các message của assistant.

```python
from langsmith import Client
from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT

client = Client()

trajectory_evaluator = create_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT,
)

def run_agent(inputs):
    return agent.invoke(inputs)["messages"]

experiment_results = client.evaluate(
    run_agent,
    data="your_dataset_name",
    evaluators=[trajectory_evaluator]
)
```

!!! tip "Mẹo"
    Để tìm hiểu thêm về cách đánh giá agent của bạn, xem [tài liệu LangSmith](https://docs.langchain.com/langsmith/pytest).
