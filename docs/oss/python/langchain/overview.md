# Tổng quan LangChain

> LangChain cung cấp `create_agent`: một agent harness tối giản, có khả năng cấu hình cao. Kết hợp đúng agent mà use case của bạn cần từ model, tool, prompt, và middleware.

**Agent = Model + Harness.** LangChain cung cấp `create_agent`: một harness tối giản, có khả năng cấu hình cao. Harness là toàn bộ phần bao quanh vòng lặp model (model loop): prompt, tool, và bất kỳ middleware nào định hình hành vi. Bắt đầu với các primitive và kết hợp đúng những gì use case của bạn cần. Hỗ trợ [OpenAI, Anthropic, Google, và nhiều provider khác](/oss/python/integrations/providers/overview).

!!! tip "LangChain so với LangGraph so với Deep Agents"
    Bắt đầu với [Deep Agents](/oss/python/deepagents/overview/) nếu muốn một agent "đầy đủ tính năng" (batteries-included) với các khả năng như tự động nén context (context compression), virtual filesystem, và khả năng sinh subagent. Deep Agents được xây dựng trên nền [agent](/oss/python/langchain/agents/) của LangChain, thứ mà bạn cũng có thể dùng trực tiếp.

    Dùng [LangChain](/oss/python/langchain/agents) (`create_agent`) khi cần một harness có khả năng tùy biến cao, dễ dàng điều chỉnh theo use case và dữ liệu của bạn.

    Dùng [LangGraph](/oss/python/langgraph/overview), framework điều phối (orchestration) cấp thấp của chúng tôi, cho các nhu cầu nâng cao kết hợp giữa workflow tất định (deterministic) và workflow agentic.

    Dùng [LangSmith](/langsmith/observability) để trace, debug, và đánh giá (evaluate) agent được xây dựng bằng bất kỳ framework nào trong số này. Làm theo [tracing quickstart](/langsmith/trace-with-langchain) để thiết lập. Chúng tôi cũng khuyến nghị bạn thiết lập [LangSmith Engine](/langsmith/engine), công cụ giám sát trace, phát hiện vấn đề, và đề xuất cách khắc phục.

## Tạo một agent

Ví dụ dưới đây minh họa cách tạo một agent LangChain đơn giản với một tool tùy chỉnh:

=== "OpenAI"
    ```python
    # pip install -qU langchain "langchain[openai]"
    from langchain.agents import create_agent

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="openai:gpt-5.5",
        tools=[get_weather],
        system_prompt="You are a helpful assistant",
    )

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
    )
    print(result["messages"][-1].content_blocks)
    ```

=== "Google Gemini"
    ```python
    # pip install -qU langchain "langchain[google-genai]"
    from langchain.agents import create_agent

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="google_genai:gemini-2.5-flash-lite",
        tools=[get_weather],
        system_prompt="You are a helpful assistant",
    )

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
    )
    print(result["messages"][-1].content_blocks)
    ```

=== "Claude (Anthropic)"
    ```python
    # pip install -qU langchain "langchain[anthropic]"
    from langchain.agents import create_agent

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="claude-sonnet-4-6",
        tools=[get_weather],
        system_prompt="You are a helpful assistant",
    )

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
    )
    print(result["messages"][-1].content_blocks)
    ```

=== "OpenRouter"
    ```python
    # pip install -qU langchain langchain-openrouter
    from langchain.agents import create_agent

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="openrouter:anthropic/claude-sonnet-4-6",
        tools=[get_weather],
        system_prompt="You are a helpful assistant",
    )

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
    )
    print(result["messages"][-1].content_blocks)
    ```

=== "Fireworks"
    ```python
    # pip install -qU langchain langchain-fireworks
    from langchain.agents import create_agent

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="fireworks:accounts/fireworks/models/qwen3p5-397b-a17b",
        tools=[get_weather],
        system_prompt="You are a helpful assistant",
    )

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
    )
    print(result["messages"][-1].content_blocks)
    ```

=== "Baseten"
    ```python
    # pip install -qU langchain langchain-baseten
    from langchain.agents import create_agent

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="baseten:zai-org/GLM-5.2",
        tools=[get_weather],
        system_prompt="You are a helpful assistant",
    )

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
    )
    print(result["messages"][-1].content_blocks)
    ```

=== "Ollama"
    ```python
    # pip install -qU langchain langchain-ollama
    from langchain.agents import create_agent

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="ollama:devstral-2",
        tools=[get_weather],
        system_prompt="You are a helpful assistant",
    )

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
    )
    print(result["messages"][-1].content_blocks)
    ```

=== "Azure"
    ```python
    # pip install -qU langchain "langchain[openai]"
    import os
    from langchain.agents import create_agent
    from langchain.chat_models import init_chat_model

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    model = init_chat_model(
        "azure_openai:gpt-5.5",
        azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"],
    )
    agent = create_agent(
        model=model,
        tools=[get_weather],
        system_prompt="You are a helpful assistant",
    )

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
    )
    print(result["messages"][-1].content_blocks)
    ```

=== "AWS Bedrock"
    ```python
    # pip install -qU langchain langchain-aws
    from langchain.agents import create_agent

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    # US cross-region inference profile; dùng global.anthropic.claude-sonnet-4-6 để route trên toàn cầu.
    agent = create_agent(
        model="bedrock_converse:us.anthropic.claude-sonnet-4-6",
        tools=[get_weather],
        system_prompt="You are a helpful assistant",
    )

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
    )
    print(result["messages"][-1].content_blocks)
    ```

=== "HuggingFace"
    ```python
    # pip install -qU langchain "langchain[huggingface]"
    from langchain.agents import create_agent

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="huggingface:microsoft/Phi-3-mini-4k-instruct",
        tools=[get_weather],
        system_prompt="You are a helpful assistant",
    )

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
    )
    print(result["messages"][-1].content_blocks)
    ```

Xem [hướng dẫn cài đặt](/oss/python/langchain/install) và [hướng dẫn bắt đầu nhanh](/oss/python/langchain/quickstart) để bắt đầu xây dựng agent và ứng dụng của riêng bạn với LangChain.

!!! tip "Mẹo"
    Dùng [LangSmith](/langsmith/observability) để trace request, debug hành vi agent, và đánh giá output. Set `LANGSMITH_TRACING=true` và API key để bắt đầu.

## Lợi ích cốt lõi

**Giao diện model chuẩn hóa** ([tìm hiểu thêm](/oss/python/langchain/models))
Dùng một interface duy nhất cho chat model, embedding, và nhiều thứ khác trên các provider. Chuyển đổi model chỉ với thay đổi code tối thiểu, giữ ứng dụng của bạn portable khi yêu cầu thay đổi.

**Harness có khả năng cấu hình cao** ([tìm hiểu thêm](/oss/python/langchain/agents))
Bắt đầu với `create_agent` như một harness tối giản và thêm dần khả năng qua middleware. Chỉ kết hợp những gì use case của bạn cần, từ guardrail, retry, cho đến routing và custom tool policy.

**Xây dựng trên nền LangGraph** ([tìm hiểu thêm](/oss/python/langgraph/overview))
Agent của LangChain được xây dựng trên nền LangGraph. Điều này cho phép chúng tôi tận dụng durable execution, hỗ trợ human-in-the-loop, persistence, và nhiều hơn nữa của LangGraph.

**Debug với LangSmith** ([tìm hiểu thêm](/langsmith/observability))
Kiểm tra trace, tool call, chuyển trạng thái (state transition), và latency ở một nơi duy nhất. Tìm ra failure mode, đánh giá chất lượng, và cải thiện hành vi agent bằng dữ liệu thực thi.
