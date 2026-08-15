# Bắt đầu nhanh

> Xây dựng agent đầu tiên của bạn chỉ trong vài phút

Hướng dẫn bắt đầu nhanh này chỉ cho bạn cách tạo một AI agent hoạt động đầy đủ chỉ trong vài phút.

!!! tip "Đang dùng AI coding assistant?"
    * Cài [LangChain Docs MCP server](/use-these-docs) để agent của bạn truy cập được tài liệu và ví dụ LangChain mới nhất.
    * Cài [LangChain Skills](https://github.com/langchain-ai/langchain-skills) để cải thiện hiệu năng agent của bạn trên các tác vụ trong hệ sinh thái LangChain.

## Cài đặt dependency

Cài các package sau để làm theo hướng dẫn:

=== "uv"
    ```bash
    uv init
    uv add langchain
    uv sync
    ```

=== "pip"
    ```bash
    pip install -U langchain
    ```

=== "venv"
    ```bash
    python3 -m venv .venv
    source .venv/bin/activate
    # Windows: .venv\Scripts\activate
    pip install -U langchain
    ```

## Thiết lập API key

Lấy API key từ [bất kỳ model provider nào được hỗ trợ](/oss/python/integrations/providers/overview) (ví dụ: Google Gemini hoặc OpenAI).

Thiết lập API key, ví dụ:

=== "OpenAI"
    ```bash
    export OPENAI_API_KEY="your-api-key"
    ```

=== "Google Gemini"
    ```bash
    export GOOGLE_API_KEY="your-api-key"
    ```

=== "Claude (Anthropic)"
    ```bash
    export ANTHROPIC_API_KEY="your-api-key"
    ```

=== "OpenRouter"
    ```bash
    export OPENROUTER_API_KEY="your-api-key"
    ```

=== "Fireworks"
    ```bash
    export FIREWORKS_API_KEY="your-api-key"
    ```

=== "Baseten"
    ```bash
    export BASETEN_API_KEY="your-api-key"
    ```

=== "Ollama"
    ```bash
    # Local: Ollama phải đang chạy (https://ollama.com)
    # Cloud: thiết lập Ollama API key cho hosted inference
    export OLLAMA_API_KEY="your-api-key"
    ```

=== "Azure"
    ```bash
    export AZURE_OPENAI_API_KEY="your-api-key"
    export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com"
    export AZURE_OPENAI_DEPLOYMENT_NAME="your-deployment"
    ```

=== "AWS Bedrock"
    ```bash
    export AWS_ACCESS_KEY_ID="your-access-key"
    export AWS_SECRET_ACCESS_KEY="your-secret-key"
    export AWS_REGION="us-east-1"
    ```

=== "HuggingFace"
    ```bash
    export HUGGINGFACEHUB_API_TOKEN="hf_..."
    ```

=== "Khác"
    Xem danh sách đầy đủ [tích hợp chat model](/oss/python/integrations/chat) được hỗ trợ.

!!! tip "Dùng LangSmith Gateway"
    [LangSmith Gateway](/langsmith/llm-gateway) route hầu hết các provider lớn qua LangSmith. Bạn có thể [dùng key provider của riêng bạn](/langsmith/llm-gateway-quickstart#2-make-a-call), hoặc dùng [Gateway Credits](/langsmith/llm-gateway-credits) để truy cập model mà không cần key provider.

## Xây dựng agent cơ bản

Bắt đầu bằng cách tạo một agent đơn giản có thể trả lời câu hỏi và gọi tool. Agent trong ví dụ này dùng language model đã chọn, một hàm thời tiết cơ bản làm tool, và một prompt đơn giản để định hướng hành vi:

=== "OpenAI"
    ```python
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
    from langchain.agents import create_agent

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

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

Khi bạn chạy code và yêu cầu agent cho biết thời tiết ở San Francisco, agent dùng input đó cùng context sẵn có.
Agent hiểu rằng bạn đang hỏi về thời tiết cho thành phố San Francisco, và vì vậy gọi tool thời tiết với tên thành phố được cung cấp.

!!! tip "Mẹo"
    Bạn có thể dùng [bất kỳ model nào được hỗ trợ](/oss/python/integrations/providers/overview) bằng cách đổi tên model và thiết lập API key phù hợp. Trace những gì đang xảy ra bên trong agent của bạn với [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-quickstart). Làm theo [tracing quickstart](/langsmith/trace-with-langchain) để thiết lập.

    Chúng tôi cũng khuyến nghị bạn thiết lập [LangSmith Engine](/langsmith/engine), công cụ giám sát trace, phát hiện vấn đề, và đề xuất cách khắc phục.

## Xây dựng agent cho tình huống thực tế

Trong ví dụ sau, bạn sẽ xây dựng một research agent có thể trả lời câu hỏi về file văn bản.
Trong quá trình đó, bạn sẽ tìm hiểu các khái niệm sau:

1. **System prompt chi tiết** để agent có hành vi tốt hơn
2. **Tạo tool** tích hợp với dữ liệu bên ngoài
3. **Cấu hình model** để có response nhất quán
4. **Bộ nhớ hội thoại (conversational memory)** cho tương tác kiểu chat
5. **Deep Agents** cho các tính năng có sẵn (built-in)
6. **Kiểm thử (testing)** agent của bạn

1. **Định nghĩa system prompt**

   System prompt định nghĩa vai trò và hành vi của agent. Giữ nó cụ thể và có thể hành động:

   ```python
   SYSTEM_PROMPT = """You are a literary data assistant.

   ## Capabilities

   - `fetch_text_from_url`: loads document text from a URL into the conversation.
   Do not guess line counts or positions—ground them in tool results from the saved file."""
   ```

2. **Tạo tool**

   [Tool](/oss/python/langchain/tools) cho phép model tương tác với hệ thống bên ngoài bằng cách gọi các hàm bạn định nghĩa.
   Tool có thể phụ thuộc vào [runtime context](/oss/python/langchain/runtime) và cũng có thể tương tác với [bộ nhớ của agent](/oss/python/langchain/short-term-memory).

   Ví dụ này dùng một tool để load một tài liệu từ một URL cho trước:

   ```python
   import urllib.error
   import urllib.request

   from langchain.tools import tool


   @tool
   def fetch_text_from_url(url: str) -> str:
       """Fetch the document from a URL.
       """
       req = urllib.request.Request(
           url,
           headers={"User-Agent": "Mozilla/5.0 (compatible; quickstart-research/1.0)"},
       )
       try:
           with urllib.request.urlopen(req, timeout=120) as resp:
               raw = resp.read()
       except urllib.error.URLError as e:
           return f"Fetch failed: {e}"
       text = raw.decode("utf-8", errors="replace")
       return text
   ```

   !!! tip "Mẹo"
       Tool nên được viết tài liệu đầy đủ: tên, mô tả, và tên tham số của nó sẽ trở thành một phần của prompt gửi cho model.
       [Decorator `@tool`](https://reference.langchain.com/python/langchain-core/tools/convert/tool) của LangChain thêm metadata và cho phép inject runtime bằng tham số `ToolRuntime`.
       Tìm hiểu thêm trong [hướng dẫn về tool](/oss/python/langchain/tools).

3. **Cấu hình model**

   Thiết lập [language model](/oss/python/langchain/models) với tham số phù hợp cho use case của bạn. Ví dụ:

   === "OpenAI"
       ```python
       from langchain.chat_models import init_chat_model

       model = init_chat_model(
           "openai:gpt-5.5",
           temperature=0.5,
           timeout=300,
           max_tokens=25000,
       )
       ```

   === "Google Gemini"
       ```python
       from langchain.chat_models import init_chat_model

       model = init_chat_model(
           "gemini-3.1-pro-preview",
           model_provider="google-genai",
           temperature=0.5,
           timeout=600,
           max_tokens=25000,
           streaming=True,
       )
       ```

   === "Claude (Anthropic)"
       ```python
       from langchain.chat_models import init_chat_model

       model = init_chat_model(
           "claude-sonnet-4-6",
           temperature=0.5,
           timeout=600,
           max_tokens=25000,
           streaming=True,
       )
       ```

   === "OpenRouter"
       ```python
       from langchain.chat_models import init_chat_model

       model = init_chat_model(
           "openrouter:anthropic/claude-sonnet-4-6",
           temperature=0.5,
           timeout=300,
           max_tokens=25000,
       )
       ```

   === "Fireworks"
       ```python
       from langchain.chat_models import init_chat_model

       model = init_chat_model(
           "fireworks:accounts/fireworks/models/qwen3p5-397b-a17b",
           temperature=0.5,
           timeout=300,
           max_tokens=25000,
       )
       ```

   === "Baseten"
       ```python
       from langchain.chat_models import init_chat_model

       model = init_chat_model(
           "baseten:zai-org/GLM-5.2",
           temperature=0.5,
           timeout=300,
           max_tokens=25000,
       )
       ```

   === "Ollama"
       ```python
       from langchain.chat_models import init_chat_model

       model = init_chat_model(
           "ollama:devstral-2",
           temperature=0.5,
           timeout=300,
           max_tokens=25000,
       )
       ```

   === "Azure"
       ```python
       import os
       from langchain.chat_models import init_chat_model

       model = init_chat_model(
           "azure_openai:gpt-5.5",
           temperature=0.5,
           timeout=300,
           max_tokens=25000,
           azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"],
       )
       ```

   === "AWS Bedrock"
       ```python
       from langchain.chat_models import init_chat_model

       model = init_chat_model(
           "us.anthropic.claude-sonnet-4-6",
           model_provider="bedrock_converse",
           temperature=0.5,
           timeout=300,
           max_tokens=25000,
       )
       ```

   === "HuggingFace"
       ```python
       from langchain.chat_models import init_chat_model

       model = init_chat_model(
           "microsoft/Phi-3-mini-4k-instruct",
           model_provider="huggingface",
           temperature=0.5,
           timeout=300,
           max_tokens=25000,
       )
       ```

   Tùy theo model và provider được chọn, tham số khởi tạo có thể khác nhau; tham khảo trang reference của chúng để biết chi tiết.

4. **Thêm bộ nhớ (memory)**

   Thêm [memory](/oss/python/langchain/short-term-memory) vào agent để duy trì state qua nhiều tương tác. Điều này cho phép agent nhớ các hội thoại và context trước đó.

   ```python
   from langgraph.checkpoint.memory import InMemorySaver

   checkpointer = InMemorySaver()
   ```

   !!! info "Thông tin"
       Trong production, hãy dùng một checkpointer bền vững (persistent), lưu lịch sử message vào database.
       Xem [Add and manage memory](/oss/python/langgraph/add-memory#manage-short-term-memory) để biết thêm chi tiết.

5. **Tạo và chạy agent**

   Giờ hãy lắp ráp agent của bạn với toàn bộ các thành phần và chạy nó.

   Có hai framework khác nhau để tạo agent: LangChain agents và deep agents.
   Cả LangChain và deep agents đều cho bạn quyền kiểm soát chi tiết với tool, memory, và nhiều thứ khác.
   Khác biệt chính giữa hai loại này là deep agents đi kèm sẵn một loạt khả năng thường hữu ích, như planning, tool filesystem, và subagent.

   Dùng deep agents khi bạn muốn khả năng tối đa với setup tối thiểu; chọn LangChain agents khi bạn cần kiểm soát chi tiết.

   Để so sánh cả hai trong bước này, cài package `deepagents`:

   === "uv"
       ```bash
       uv add deepagents
       ```

   === "pip"
       ```bash
       pip install -U deepagents
       ```

   !!! warning "Cảnh báo"
       Vì code này gọi model với toàn bộ văn bản của The Great Gatsby, nó dùng lượng token lớn.

       Bạn có thể xem output ví dụ ở bước tiếp theo.

   Hãy thử cả hai:

   ```python
   from langchain.agents import create_agent
   from deepagents import create_deep_agent

   agent = create_agent(
       model=model,
       tools=[fetch_text_from_url],
       system_prompt=SYSTEM_PROMPT,
       checkpointer=checkpointer,
   )

   deep_agent = create_deep_agent(
       model=model,
       tools=[fetch_text_from_url],
       system_prompt=SYSTEM_PROMPT,
       checkpointer=checkpointer,
   )

   content = f"""Project Gutenberg hosts a full plain-text copy of F. Scott Fitzgerald's The Great Gatsby.
   URL: https://www.gutenberg.org/files/64317/64317-0.txt

   Answer as much as you can:

   1) How many lines in the complete Gutenberg file contain the substring `Gatsby` (count lines, not occurrences within a line, each line ends with a line break).
   2) The 1-based line number of the first line in the file that contains `Daisy`.
   3) A two-sentence neutral synopsis.

   Do your best on (1) and (2). If at any point you realize you cannot **verify** an exact answer with
   your available tools and reasoning, do not fabricate numbers: use `null` for that field and spell out
   the limitation in `how_you_computed_counts`. If you encounter any errors please report what the error was and what the error message was."""

   agent_result = agent.invoke(
       {"messages": [{"role": "user", "content": content}]},
       config={"configurable": {"thread_id": "great-gatsby-lc"}},
   )
   deep_agent_result = deep_agent.invoke(
       {"messages": [{"role": "user", "content": content}]},
       config={"configurable": {"thread_id": "great-gatsby-da"}},
   )
   print(agent_result["messages"][-1].content_blocks)
   print("\n")
   print(deep_agent_result["messages"][-1].content_blocks)
   ```

   **Code ví dụ đầy đủ**

   ```python
   import urllib.error
   import urllib.request

   from langchain.agents import create_agent
   from deepagents import create_deep_agent
   from langchain.chat_models import init_chat_model
   from langchain.tools import tool
   from langgraph.checkpoint.memory import InMemorySaver

   SYSTEM_PROMPT = """You are a literary data assistant.

   ## Capabilities

   - `fetch_text_from_url`: loads document text from a URL into the conversation.
   Do not guess line counts or positions—ground them in tool results from the saved file."""


   @tool
   def fetch_text_from_url(url: str) -> str:
       """Fetch the document from a URL.
       """
       req = urllib.request.Request(
           url,
           headers={"User-Agent": "Mozilla/5.0 (compatible; quickstart-research/1.0)"},
       )
       try:
           with urllib.request.urlopen(req, timeout=120) as resp:
               raw = resp.read()
       except urllib.error.URLError as e:
           return f"Fetch failed: {e}"
       text = raw.decode("utf-8", errors="replace")
       return text


   model = init_chat_model(
       "gemini-3.1-pro-preview",
       model_provider="google-genai",
       temperature=0.5,
       timeout=600,
       max_tokens=25000,
       streaming=True,
   )

   checkpointer = InMemorySaver()

   agent = create_agent(
       model=model,
       tools=[fetch_text_from_url],
       system_prompt=SYSTEM_PROMPT,
       checkpointer=checkpointer,
   )

   deep_agent = create_deep_agent(
       model=model,
       tools=[fetch_text_from_url],
       system_prompt=SYSTEM_PROMPT,
       checkpointer=checkpointer,
   )

   content = f"""Project Gutenberg hosts a full plain-text copy of F. Scott Fitzgerald's The Great Gatsby.
   URL: https://www.gutenberg.org/files/64317/64317-0.txt

   Answer as much as you can:

   1) How many lines in the complete Gutenberg file contain the substring `Gatsby` (count lines, not occurrences within a line, each line ends with a line break).
   2) The 1-based line number of the first line in the file that contains `Daisy`.
   3) A two-sentence neutral synopsis.

   Do your best on (1) and (2). If at any point you realize you cannot **verify** an exact answer with
   your available tools and reasoning, do not fabricate numbers: use `null` for that field and spell out
   the limitation in `how_you_computed_counts`. If you encounter any errors please report what the error was and what the error message was."""

   agent_result = agent.invoke(
       {"messages": [{"role": "user", "content": content}]},
       config={"configurable": {"thread_id": "great-gatsby-lc"}},
   )
   deep_agent_result = deep_agent.invoke(
       {"messages": [{"role": "user", "content": content}]},
       config={"configurable": {"thread_id": "great-gatsby-da"}},
   )
   print(agent_result["messages"][-1].content_blocks)
   print("\n")
   print(deep_agent_result["messages"][-1].content_blocks)
   ```

6. **Xem lại kết quả**

   Kết quả sẽ khác nhau tùy theo model và lần chạy.

   === "LangChain agents"
       ```txt
       **1) Number of lines containing `Gatsby`:** `null`

       **2) First line containing `Daisy`:** `null`

       **3) Synopsis:**
       The Great Gatsby follows the mysterious millionaire Jay Gatsby and his obsession with reuniting with his former lover, Daisy Buchanan, as narrated by his neighbor Nick Carraway. Set against the backdrop of the Roaring Twenties on Long Island, the novel explores themes of wealth, class, and the elusive nature of the American Dream.

       **how_you_computed_counts:**
       I successfully fetched the full text of the eBook using the `fetch_text_from_url` tool. However, because I do not have access to a code execution environment (like Python) or text-processing tools (like `grep`), I cannot deterministically split the text by line breaks, iterate through the thousands of lines, and verify the exact line numbers or match counts. LLMs cannot reliably perform exact line-counting or indexing over massive texts within their context window without external computational tools. As instructed, rather than fabricating or guessing a number, I have output `null` for the exact counts and positions.
       ```

   === "Deep agents"
       ```txt
       Based on the text fetched directly from the Gutenberg URL and analyzed using filesystem search tools, here are the answers to your questions:

       **1) Lines containing the substring `Gatsby`**
       **258** lines contain the exact substring `Gatsby`.

       **2) First line containing `Daisy`**
       Line **181** is the first line in the file that contains the exact substring `Daisy`.
       *(For context, the line reads: "Buchanans. Daisy was my second cousin once removed, and I'd known Tom")*

       **3) Two-sentence neutral synopsis**
       *The Great Gatsby* follows the mysterious millionaire Jay Gatsby and his obsessive pursuit to reunite with his former lover, Daisy Buchanan, in 1920s Long Island. The story is narrated by Nick Carraway, who observes the tragic consequences of Gatsby's relentless ambition and the shallow materialism of the era's wealthy elite.

       ---

       **How counts were computed:**
       When fetching the document from the URL, the file was too large for the standard output and was automatically saved to the local filesystem by the system (`/large_tool_results/x246ax2x`). I then used the `grep` tool to search the saved file for the exact literal substrings `Gatsby` and `Daisy`. The `grep` tool returned every matching line along with its 1-based line number. I manually counted the exact number of lines returned for `Gatsby` (which totaled 258) and identified the first line number returned for `Daisy` (which was 181). I also verified there were no uppercase variations (`GATSBY` or `DAISY`) that would have been missed. No errors were encountered during this process.
       ```

   Nếu bạn nhìn vào output ở cả hai tab, bạn sẽ thấy LangChain agent có đưa ra câu trả lời nhưng chỉ là ước lượng. Agent thiếu tool để trả lời chính xác câu hỏi này. Bạn cũng có thể gặp lỗi vì prompt quá dài.

   Ngược lại, deep agent có thể:

   1. **Lập kế hoạch cách tiếp cận** bằng tool có sẵn [`write_todos`](/oss/python/deepagents/harness#task-planning) để chia nhỏ tác vụ nghiên cứu.
   2. **Load file** bằng cách gọi tool `fetch_text_from_url` để thu thập thông tin.
   3. **Quản lý context** bằng cách dùng tool filesystem ([`grep`](/oss/python/deepagents/harness#virtual-filesystem-access) và [`read_file`](/oss/python/deepagents/harness#virtual-filesystem-access)).
   4. **Sinh subagent** khi cần để giao các subtask phức tạp cho subagent chuyên biệt.

   Với LangChain agents, bạn phải tự triển khai thêm nhiều khả năng để đạt mức độ tương tự, nhưng có thể tùy chỉnh dần theo nhu cầu.

## Trace các lệnh gọi agent

Hầu hết ứng dụng thú vị bạn xây dựng với LangChain đều thực hiện nhiều lệnh gọi tới LLM. Khi các ứng dụng này trở nên phức tạp hơn, việc có thể kiểm tra chính xác những gì đang xảy ra bên trong agent trở nên quan trọng. Cách tốt nhất để làm điều này là dùng [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-quickstart).

Đăng ký tài khoản [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-quickstart) và thiết lập các biến sau để bắt đầu log trace:

```shell
export LANGSMITH_TRACING="true"
export LANGSMITH_API_KEY="..."
```

Sau khi thiết lập, chạy lại script của bạn rồi kiểm tra những gì xảy ra trong các lệnh gọi agent trên [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-quickstart).

!!! tip "Mẹo"
    Để tìm hiểu thêm về việc trace agent của bạn với LangSmith, xem [tài liệu LangSmith](/langsmith/trace-with-langchain).

    Chúng tôi cũng khuyến nghị bạn thiết lập [LangSmith Engine](/langsmith/engine), công cụ giám sát trace, phát hiện vấn đề, và đề xuất cách khắc phục.

## Bước tiếp theo

Giờ bạn đã có agent có thể:

* **Hiểu context** và nhớ hội thoại
* **Dùng tool** một cách thông minh
* **Cung cấp response có cấu trúc** theo định dạng nhất quán
* **Xử lý thông tin đặc thù theo user** thông qua context
* **Duy trì state hội thoại** qua nhiều tương tác
* **Lập kế hoạch, nghiên cứu, và tổng hợp** (chỉ deep agents)

Tiếp tục với:

* **LangChain agents**: [Add and manage memory](/oss/python/langgraph/add-memory#manage-short-term-memory), [deploy lên production](/oss/python/langgraph/deploy)
* **Deep Agents**: [Tùy chọn tùy chỉnh](/oss/python/deepagents/customization), [bộ nhớ bền vững](/oss/python/deepagents/memory), [deploy lên production](/oss/python/langgraph/deploy)
