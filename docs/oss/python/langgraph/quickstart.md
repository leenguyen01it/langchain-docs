# Quickstart

Quickstart này minh hoạ cách xây dựng một agent máy tính (calculator agent) bằng LangGraph Graph API hoặc Functional API.

!!! tip "Đang dùng AI coding assistant?"
    * Cài [LangChain Docs MCP server](https://docs.langchain.com/use-these-docs) để agent của bạn truy cập được tài liệu và ví dụ LangChain mới nhất.
    * Cài [LangChain Skills](https://github.com/langchain-ai/langchain-skills) để cải thiện hiệu suất agent của bạn với các tác vụ trong hệ sinh thái LangChain.

* [Dùng Graph API](#use-the-graph-api) nếu bạn muốn định nghĩa agent dưới dạng một graph gồm các node và cạnh.
* [Dùng Functional API](#use-the-functional-api) nếu bạn muốn định nghĩa agent bằng một hàm duy nhất.

Để biết thông tin khái niệm, xem [Tổng quan Graph API](./graph-api.md) và [Tổng quan Functional API](./functional-api.md).

!!! info ""
    Với ví dụ này, bạn cần thiết lập tài khoản [Claude (Anthropic)](https://www.anthropic.com/) và lấy API key. Sau đó, đặt biến môi trường `ANTHROPIC_API_KEY` trong terminal của bạn. Xem [tích hợp chat model](https://docs.langchain.com/oss/python/integrations/chat) để biết tất cả nhà cung cấp có sẵn. Nếu bạn dùng [LangSmith Gateway](https://docs.langchain.com/langsmith/llm-gateway), bạn có thể [dùng API key nhà cung cấp của riêng mình](https://docs.langchain.com/langsmith/llm-gateway-quickstart) hoặc dùng [Gateway Credits](https://docs.langchain.com/langsmith/llm-gateway-credits) để truy cập model mà không cần key nhà cung cấp.

=== "Dùng Graph API"

    ## 1. Định nghĩa tool và model

    Trong ví dụ này, ta dùng model Claude Sonnet 4.5 và định nghĩa các tool cho phép cộng, nhân và chia.

    ```python
    from langchain.tools import tool
    from langchain.chat_models import init_chat_model


    model = init_chat_model(
        "claude-sonnet-4-6",
        temperature=0
    )


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


    # Bổ sung tool cho LLM
    tools = [add, multiply, divide]
    tools_by_name = {tool.name: tool for tool in tools}
    model_with_tools = model.bind_tools(tools)
    ```

    ## 2. Định nghĩa state

    State của graph dùng để lưu tin nhắn và số lần gọi LLM.

    !!! tip ""
        State trong LangGraph tồn tại xuyên suốt quá trình thực thi của agent.

        Kiểu `Annotated` với `operator.add` đảm bảo tin nhắn mới được nối thêm vào danh sách hiện có thay vì thay thế nó.

    ```python
    from langchain.messages import AnyMessage
    from typing_extensions import TypedDict, Annotated
    import operator


    class MessagesState(TypedDict):
        messages: Annotated[list[AnyMessage], operator.add]
        llm_calls: int
    ```

    ## 3. Định nghĩa model node

    Model node dùng để gọi LLM và quyết định có gọi tool hay không.

    ```python
    from langchain.messages import SystemMessage


    def llm_call(state: dict):
        """LLM decides whether to call a tool or not"""

        return {
            "messages": [
                model_with_tools.invoke(
                    [
                        SystemMessage(
                            content="You are a helpful assistant tasked with performing arithmetic on a set of inputs."
                        )
                    ]
                    + state["messages"]
                )
            ],
            "llm_calls": state.get('llm_calls', 0) + 1
        }
    ```

    ## 4. Định nghĩa tool node

    Tool node dùng để gọi các tool và trả về kết quả.

    ```python
    from langchain.messages import ToolMessage


    def tool_node(state: dict):
        """Performs the tool call"""

        result = []
        for tool_call in state["messages"][-1].tool_calls:
            tool = tools_by_name[tool_call["name"]]
            observation = tool.invoke(tool_call["args"])
            result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
        return {"messages": result}
    ```

    ## 5. Định nghĩa logic kết thúc

    Hàm conditional edge dùng để định tuyến tới tool node hoặc kết thúc, dựa trên việc LLM có gọi tool hay không.

    ```python
    from typing import Literal
    from langgraph.graph import StateGraph, START, END


    def should_continue(state: MessagesState) -> Literal["tool_node", END]:
        """Decide if we should continue the loop or stop based upon whether the LLM made a tool call"""

        messages = state["messages"]
        last_message = messages[-1]

        # If the LLM makes a tool call, then perform an action
        if last_message.tool_calls:
            return "tool_node"

        # Otherwise, we stop (reply to the user)
        return END
    ```

    ## 6. Xây dựng và compile agent

    Agent được xây dựng bằng class [`StateGraph`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) và compile bằng phương thức [`compile`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/compile).

    ```python
    # Build workflow
    agent_builder = StateGraph(MessagesState)

    # Add nodes
    agent_builder.add_node("llm_call", llm_call)
    agent_builder.add_node("tool_node", tool_node)

    # Add edges to connect nodes
    agent_builder.add_edge(START, "llm_call")
    agent_builder.add_conditional_edges(
        "llm_call",
        should_continue,
        ["tool_node", END]
    )
    agent_builder.add_edge("tool_node", "llm_call")

    # Compile the agent
    agent = agent_builder.compile()

    # Show the agent
    from IPython.display import Image, display
    display(Image(agent.get_graph(xray=True).draw_mermaid_png()))

    # Invoke
    from langchain.messages import HumanMessage
    messages = [HumanMessage(content="Add 3 and 4.")]
    messages = agent.invoke({"messages": messages})
    for m in messages["messages"]:
        m.pretty_print()
    ```

    !!! tip ""
        Trace và debug agent của bạn với [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langgraph-quickstart). Làm theo [quickstart tracing](https://docs.langchain.com/langsmith/trace-with-langgraph) để thiết lập. Khi sẵn sàng cho production, xem [Deploy](https://docs.langchain.com/langsmith/deployment) để biết các lựa chọn hosting.

        Chúng tôi khuyến nghị bạn cũng thiết lập [LangSmith Engine](https://docs.langchain.com/langsmith/engine), công cụ giám sát trace của bạn, phát hiện vấn đề và đề xuất cách khắc phục.

    Chúc mừng! Bạn đã xây dựng agent đầu tiên bằng LangGraph Graph API.

    ??? note "Ví dụ code đầy đủ"
        ```python
        # Step 1: Define tools and model

        from langchain.tools import tool
        from langchain.chat_models import init_chat_model


        model = init_chat_model(
            "claude-sonnet-4-6",
            temperature=0
        )


        # Define tools
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


        # Augment the LLM with tools
        tools = [add, multiply, divide]
        tools_by_name = {tool.name: tool for tool in tools}
        model_with_tools = model.bind_tools(tools)

        # Step 2: Define state

        from langchain.messages import AnyMessage
        from typing_extensions import TypedDict, Annotated
        import operator


        class MessagesState(TypedDict):
            messages: Annotated[list[AnyMessage], operator.add]
            llm_calls: int

        # Step 3: Define model node
        from langchain.messages import SystemMessage


        def llm_call(state: MessagesState):
            """LLM decides whether to call a tool or not"""

            return {
                "messages": [
                    model_with_tools.invoke(
                        [
                            SystemMessage(
                                content="You are a helpful assistant tasked with performing arithmetic on a set of inputs."
                            )
                        ]
                        + state["messages"]
                    )
                ],
                "llm_calls": state.get('llm_calls', 0) + 1
            }


        # Step 4: Define tool node

        from langchain.messages import ToolMessage


        def tool_node(state: MessagesState):
            """Performs the tool call"""

            result = []
            for tool_call in state["messages"][-1].tool_calls:
                tool = tools_by_name[tool_call["name"]]
                observation = tool.invoke(tool_call["args"])
                result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
            return {"messages": result}

        # Step 5: Define logic to determine whether to end

        from typing import Literal
        from langgraph.graph import StateGraph, START, END


        # Conditional edge function to route to the tool node or end based upon whether the LLM made a tool call
        def should_continue(state: MessagesState) -> Literal["tool_node", END]:
            """Decide if we should continue the loop or stop based upon whether the LLM made a tool call"""

            messages = state["messages"]
            last_message = messages[-1]

            # If the LLM makes a tool call, then perform an action
            if last_message.tool_calls:
                return "tool_node"

            # Otherwise, we stop (reply to the user)
            return END

        # Step 6: Build agent

        # Build workflow
        agent_builder = StateGraph(MessagesState)

        # Add nodes
        agent_builder.add_node("llm_call", llm_call)
        agent_builder.add_node("tool_node", tool_node)

        # Add edges to connect nodes
        agent_builder.add_edge(START, "llm_call")
        agent_builder.add_conditional_edges(
            "llm_call",
            should_continue,
            ["tool_node", END]
        )
        agent_builder.add_edge("tool_node", "llm_call")

        # Compile the agent
        agent = agent_builder.compile()


        from IPython.display import Image, display
        # Show the agent
        display(Image(agent.get_graph(xray=True).draw_mermaid_png()))

        # Invoke
        from langchain.messages import HumanMessage
        messages = [HumanMessage(content="Add 3 and 4.")]
        messages = agent.invoke({"messages": messages})
        for m in messages["messages"]:
            m.pretty_print()

        ```

=== "Dùng Functional API"

    ## 1. Định nghĩa tool và model

    Trong ví dụ này, ta dùng model Claude Sonnet 4.5 và định nghĩa các tool cho phép cộng, nhân và chia.

    ```python
    from langchain.tools import tool
    from langchain.chat_models import init_chat_model


    model = init_chat_model(
        "claude-sonnet-4-6",
        temperature=0
    )


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


    # Bổ sung tool cho LLM
    tools = [add, multiply, divide]
    tools_by_name = {tool.name: tool for tool in tools}
    model_with_tools = model.bind_tools(tools)

    from langgraph.graph import add_messages
    from langchain.messages import (
        SystemMessage,
        HumanMessage,
        ToolCall,
    )
    from langchain_core.messages import BaseMessage
    from langgraph.func import entrypoint, task
    ```

    ## 2. Định nghĩa model node

    Model node dùng để gọi LLM và quyết định có gọi tool hay không.

    !!! tip ""
        Decorator [`@task`](https://reference.langchain.com/python/langgraph/func/task) đánh dấu một hàm là một task có thể được thực thi như một phần của agent. Task có thể được gọi đồng bộ hoặc bất đồng bộ bên trong hàm entrypoint của bạn.

    ```python
    @task
    def call_llm(messages: list[BaseMessage]):
        """LLM decides whether to call a tool or not"""
        return model_with_tools.invoke(
            [
                SystemMessage(
                    content="You are a helpful assistant tasked with performing arithmetic on a set of inputs."
                )
            ]
            + messages
        )
    ```

    ## 3. Định nghĩa tool node

    Tool node dùng để gọi các tool và trả về kết quả.

    ```python
    @task
    def call_tool(tool_call: ToolCall):
        """Performs the tool call"""
        tool = tools_by_name[tool_call["name"]]
        return tool.invoke(tool_call)

    ```

    ## 4. Định nghĩa agent

    Agent được xây dựng bằng hàm [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/entrypoint).

    !!! note ""
        Trong Functional API, thay vì định nghĩa node và cạnh một cách tường minh, bạn viết logic control flow chuẩn (vòng lặp, điều kiện) bên trong một hàm duy nhất.

    ```python
    @entrypoint()
    def agent(messages: list[BaseMessage]):
        model_response = call_llm(messages).result()

        while True:
            if not model_response.tool_calls:
                break

            # Execute tools
            tool_result_futures = [
                call_tool(tool_call) for tool_call in model_response.tool_calls
            ]
            tool_results = [fut.result() for fut in tool_result_futures]
            messages = add_messages(messages, [model_response, *tool_results])
            model_response = call_llm(messages).result()

        messages = add_messages(messages, model_response)
        return messages

    # Invoke
    messages = [HumanMessage(content="Add 3 and 4.")]
    stream = agent.stream_events(messages, version="v3")
    for snapshot in stream.values:
        print(snapshot)
        print("\n")
    ```

    !!! tip ""
        Trace và debug agent của bạn với [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langgraph-quickstart). Làm theo [quickstart tracing](https://docs.langchain.com/langsmith/trace-with-langgraph) để thiết lập. Khi sẵn sàng cho production, xem [Deploy](https://docs.langchain.com/langsmith/deployment) để biết các lựa chọn hosting.

        Chúng tôi khuyến nghị bạn cũng thiết lập [LangSmith Engine](https://docs.langchain.com/langsmith/engine), công cụ giám sát trace của bạn, phát hiện vấn đề và đề xuất cách khắc phục.

    Chúc mừng! Bạn đã xây dựng agent đầu tiên bằng LangGraph Functional API.

    ??? note "Ví dụ code đầy đủ"
        ```python
        # Step 1: Define tools and model

        from langchain.tools import tool
        from langchain.chat_models import init_chat_model


        model = init_chat_model(
            "claude-sonnet-4-6",
            temperature=0
        )


        # Define tools
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


        # Augment the LLM with tools
        tools = [add, multiply, divide]
        tools_by_name = {tool.name: tool for tool in tools}
        model_with_tools = model.bind_tools(tools)

        from langgraph.graph import add_messages
        from langchain.messages import (
            SystemMessage,
            HumanMessage,
            ToolCall,
        )
        from langchain_core.messages import BaseMessage
        from langgraph.func import entrypoint, task


        # Step 2: Define model node

        @task
        def call_llm(messages: list[BaseMessage]):
            """LLM decides whether to call a tool or not"""
            return model_with_tools.invoke(
                [
                    SystemMessage(
                        content="You are a helpful assistant tasked with performing arithmetic on a set of inputs."
                    )
                ]
                + messages
            )


        # Step 3: Define tool node

        @task
        def call_tool(tool_call: ToolCall):
            """Performs the tool call"""
            tool = tools_by_name[tool_call["name"]]
            return tool.invoke(tool_call)


        # Step 4: Define agent

        @entrypoint()
        def agent(messages: list[BaseMessage]):
            model_response = call_llm(messages).result()

            while True:
                if not model_response.tool_calls:
                    break

                # Execute tools
                tool_result_futures = [
                    call_tool(tool_call) for tool_call in model_response.tool_calls
                ]
                tool_results = [fut.result() for fut in tool_result_futures]
                messages = add_messages(messages, [model_response, *tool_results])
                model_response = call_llm(messages).result()

            messages = add_messages(messages, model_response)
            return messages

        # Invoke
        messages = [HumanMessage(content="Add 3 and 4.")]
        stream = agent.stream_events(messages, version="v3")
        for snapshot in stream.values:
            print(snapshot)
            print("\n")
        ```

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/quickstart.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
