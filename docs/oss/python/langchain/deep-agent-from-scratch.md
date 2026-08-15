# Xây dựng agent phân tích dữ liệu từ đầu

> Xây dựng từng bước một agent phân tích dữ liệu bằng create_agent và middleware của Deep Agents.

Hướng dẫn này xây dựng một agent phân tích dữ liệu từ các nguyên lý cơ bản (first principles), dùng `create_agent` và middleware của Deep Agents.

Cả `create_agent` và `create_deep_agent` đều cho bạn khả năng kiểm soát chi tiết đối với tool, memory, và nhiều hơn nữa.
Khác biệt chính giữa hai cái là Deep Agents đi kèm sẵn một loạt khả năng thường dùng, như lập kế hoạch, tool filesystem, và subagent.
Nếu harness mặc định của Deep Agents không phù hợp nhu cầu của bạn, hướng dẫn này chỉ cho bạn cách bắt đầu với `create_agent` và lắp ráp harness từng phần một, để bạn thấy chính xác mỗi thành phần thêm vào cái gì và chỉ lắp những gì use case của bạn cần.

Làm theo hướng dẫn này để xây một agent có thể:

1. Nhận một file CSV để phân tích
2. Viết và thực thi code Python trong một sandbox cô lập
3. Uỷ quyền công việc visualization cho một subagent chuyên biệt
4. Nạp các pattern phân tích dữ liệu từ một skill file

Stack cuối cùng phản ánh đúng những gì `create_deep_agent` lắp ráp theo mặc định.

## Bạn sẽ học được gì

Mỗi bước thêm một khả năng vào cùng một agent phân tích dữ liệu:

| Bước | Vấn đề nếu không có | Bạn thêm gì |
| --- | --- | --- |
| Agent tối giản | (không có) | Vòng lặp cơ bản: model + tool, chưa có harness |
| Sandbox + filesystem | Agent không đọc được CSV hay chạy được Python | [Backend](https://docs.langchain.com/oss/python/deepagents/backends) cô lập + tool file và execute |
| Summarization | Session dài chạm giới hạn context | Nén lịch sử tự động |
| Skills | Quy tắc chuyên biệt làm phình system prompt | Kiến thức theo nhu cầu qua [progressive disclosure](multi-agent/skills-sql-assistant.md) |
| Subagent | Việc lặp lại biểu đồ chiếm chỗ thread chính | Worker cô lập + uỷ quyền song song |

## Thiết lập

1. **Cài package**

    Cài các package cho tutorial này:

    ```bash
    pip install deepagents langsmith
    ```

2. **Thiết lập API key LangSmith**

    Tutorial này dùng [`LangSmithSandbox`](https://reference.langchain.com/python/deepagents/backends/langsmith/LangSmithSandbox), thứ cấp phát sandbox thông qua `SandboxClient`. Client đó xác thực với LangSmith bằng `LANGSMITH_API_KEY` từ môi trường của bạn, nên cần một API key để chạy tutorial. Thiết lập LangSmith cũng cho phép bạn xem trace của những gì xảy ra khi agent chạy.

    1. [Đăng ký tài khoản miễn phí](https://smith.langchain.com). Bạn có thể dùng Google, GitHub, hoặc email.
    2. [Tạo một API key](https://docs.langchain.com/langsmith/create-account-api-key) trong **Settings → API Keys**.
    3. Export API key LangSmith:

        ```bash
        export LANGSMITH_API_KEY=...
        ```

    4. Bật tracing để kiểm tra tool call, các bước middleware, và uỷ quyền subagent khi bạn thêm từng phần:

        ```bash
        export LANGSMITH_TRACING=true
        ```

3. **Thêm API key của model provider**

    Export API key cho model provider bạn dùng trong các đoạn code mẫu:

    === "OpenAI"
        ```bash
        export OPENAI_API_KEY=...
        ```

    === "Anthropic"
        ```bash
        export ANTHROPIC_API_KEY=...
        ```

    !!! note "Ghi chú"
        Bản gốc còn có thêm tab cho Google, OpenRouter, Fireworks, Baseten, và Ollama, chỉ khác tên biến môi trường API key. Từ đây trở đi, các CodeGroup nhiều provider trong trang này được rút gọn còn 2 tab tiêu biểu (OpenAI, Anthropic).

## Xây dựng agent

## Tạo agent tối giản

Một agent phân tích dữ liệu cần nhiều hơn một vòng lặp chat, nhưng để bắt đầu, hãy khởi đầu với baseline: chỉ một model và một vòng lặp.

Dùng [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) và chỉ định model bạn muốn dùng:

=== "OpenAI"
    ```python
    from langchain.agents import create_agent

    agent = create_agent(model="openai:gpt-5.5", tools=[])
    ```

=== "Anthropic"
    ```python
    from langchain.agents import create_agent

    agent = create_agent(model="anthropic:claude-sonnet-4-6", tools=[])
    ```

Đoạn này chạy được, nhưng agent chưa có filesystem và không có cách nào thực thi code. Nếu bạn yêu cầu nó phân tích một CSV, nó chỉ có thể đoán từ prompt. Các bước tiếp theo thêm khả năng truy cập file và thực thi code thực sự.

## Thêm sandbox backend

Để phân tích dữ liệu hiệu quả, agent cần chạy code trên file. Điều này cần hai thứ:

* Một [sandbox](https://docs.langchain.com/oss/python/deepagents/sandboxes) cô lập nơi agent có thể đặt file và chạy code trên các file đó mà không cho agent quyền truy cập máy host của bạn.
* Một [backend](https://docs.langchain.com/oss/python/deepagents/backends) cung cấp các tool filesystem để làm việc với sandbox (`read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep`) thông qua [`FilesystemMiddleware`](https://reference.langchain.com/python/deepagents/middleware/filesystem/FilesystemMiddleware). Vì backend `LangSmithSandbox` triển khai sandbox protocol, [`FilesystemMiddleware`](https://reference.langchain.com/python/deepagents/middleware/filesystem/FilesystemMiddleware) cũng thêm tool `execute`, cho phép agent chạy lệnh shell.

[`LangSmithSandbox`](https://reference.langchain.com/python/deepagents/backends/langsmith/LangSmithSandbox) là nơi file tồn tại và lệnh chạy. [`FilesystemMiddleware`](https://reference.langchain.com/python/deepagents/middleware/filesystem/FilesystemMiddleware) là thứ expose môi trường đó cho model dưới dạng tool. Cùng một middleware hoạt động với backend khác nếu bạn đổi backend sau này.

[`LangSmithSandbox`](https://reference.langchain.com/python/deepagents/backends/langsmith/LangSmithSandbox) cấp cho agent một môi trường cô lập với filesystem và một tool `execute` để chạy lệnh shell. Với nó, agent có thể cài package, viết script, và chạy chúng mà không đụng tới host. Để khởi động từ một image tuỳ chỉnh thay vì runtime mặc định, truyền `snapshot_name` hoặc `snapshot_id` vào `create_sandbox()`; xem [Sandbox snapshots](https://docs.langchain.com/langsmith/sandbox-snapshots).

Thay agent từ bước trước bằng một agent có [`FilesystemMiddleware`](https://reference.langchain.com/python/deepagents/middleware/filesystem/FilesystemMiddleware):

=== "OpenAI"
    ```python
    from langchain.agents import create_agent
    from deepagents.backends.langsmith import LangSmithSandbox
    from deepagents.middleware import FilesystemMiddleware
    from langsmith.sandbox import SandboxClient

    client = SandboxClient()
    sandbox = None
    sandbox = client.create_sandbox(name="langchain-docs", snapshot_name="docs-test-ci")
    backend = LangSmithSandbox(sandbox=sandbox)

    agent = create_agent(
        model="openai:gpt-5.5",
        tools=[],
        middleware=[FilesystemMiddleware(backend=backend)],
    )
    ```

=== "Anthropic"
    ```python
    from langchain.agents import create_agent
    from deepagents.backends.langsmith import LangSmithSandbox
    from deepagents.middleware import FilesystemMiddleware
    from langsmith.sandbox import SandboxClient

    client = SandboxClient()
    sandbox = None
    sandbox = client.create_sandbox(name="langchain-docs", snapshot_name="docs-test-ci")
    backend = LangSmithSandbox(sandbox=sandbox)

    agent = create_agent(
        model="anthropic:claude-sonnet-4-6",
        tools=[],
        middleware=[FilesystemMiddleware(backend=backend)],
    )
    ```

Filesystem của sandbox tách biệt với máy tính của bạn. Bạn phải upload các file cần thiết lên đó trước khi invoke agent:

```python
import csv
import io

rows = [
    ["Date", "Product", "Units", "Revenue"],
    ["2025-08-01", "Widget A", 10, 250],
    ["2025-08-02", "Widget B", 5, 125],
    ["2025-08-03", "Widget A", 7, 175],
    ["2025-08-04", "Widget C", 3, 90],
]
buf = io.StringIO()
csv.writer(buf).writerows(rows)
backend.upload_files([("/sales.csv", buf.getvalue().encode())])

upload_stream = agent.stream_events(
    {
        "messages": [
            {
                "role": "user",
                "content": (
                    "Read /sales.csv and summarize total revenue by product in one "
                    "sentence. Do not run shell commands."
                ),
            }
        ]
    },
    version="v3",
    config={"recursion_limit": 8},
)
for item in upload_stream.messages:
    print(item.text)
upload_stream.output
```

!!! note "Ghi chú"
    Với [`LangSmithSandbox`](https://reference.langchain.com/python/deepagents/backends/langsmith/LangSmithSandbox), đường dẫn upload phải là đường dẫn POSIX tuyệt đối (ví dụ, `/sales.csv`). Đường dẫn tương đối như `sales.csv` sẽ bị từ chối với lỗi `invalid_path` và file sẽ không được ghi vào sandbox.

Gộp code từ các bước trước vào một script và chạy:

```bash
python analyze_sales.py
```

Ở lần chạy đầu tiên, LangSmith sẽ cấp phát một sandbox (có thể mất vài giây). Script upload `sales.csv`, stream lần chạy của agent, và in các assistant message khi chúng tới. Bạn sẽ thấy một phân tích về dữ liệu bán hàng mẫu: doanh thu theo từng sản phẩm, widget nào bán chạy nhất, và ghi chú xu hướng ngắn gọn. Nội dung chính xác sẽ khác nhau tuỳ theo lần chạy model.

Mở lần chạy trong [LangSmith](https://smith.langchain.com/) và quan sát agent dùng tool filesystem (`read_file`, và `execute` nếu nó chạy Python trong sandbox) trước khi phản hồi.

## Thêm quản lý context

Sau bước 2, mọi kết quả tool đều nằm trong lịch sử message. Một phiên phân tích thực tế (nhiều biểu đồ, script lỗi, output `read_file` lớn) sẽ nhanh chóng lấp đầy context window.

[`SummarizationMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/SummarizationMiddleware) nén các lượt cũ hơn khi lịch sử phình quá lớn, để agent tiếp tục hoạt động mà bạn không phải cắt message thủ công. Điều này ít quan trọng hơn với câu hỏi `sales.csv` đầu tiên và quan trọng hơn với các câu hỏi tiếp theo như "Now segment by product and plot monthly trends."

Cập nhật agent của bạn từ bước 2 bằng cách thêm [`SummarizationMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/SummarizationMiddleware) vào danh sách middleware:

=== "OpenAI"
    ```python
    from deepagents.middleware import FilesystemMiddleware, SummarizationMiddleware

    model = "openai:gpt-5.5"

    agent = create_agent(
        model=model,
        tools=[],
        middleware=[
            FilesystemMiddleware(backend=backend),
            SummarizationMiddleware(model=model, backend=backend),
        ],
    )
    ```

=== "Anthropic"
    ```python
    from deepagents.middleware import FilesystemMiddleware, SummarizationMiddleware

    model = "anthropic:claude-sonnet-4-6"

    agent = create_agent(
        model=model,
        tools=[],
        middleware=[
            FilesystemMiddleware(backend=backend),
            SummarizationMiddleware(model=model, backend=backend),
        ],
    )
    ```

Chạy một phiên nhiều lượt để thấy summarization hoạt động. Sau phân tích ban đầu, hỏi các câu tiếp theo kích hoạt thêm lần đọc file hoặc chạy script. Trong LangSmith, tìm một bước summarization trước các lần gọi model sau đó. Để biết thêm thông tin, xem [Context engineering](context-engineering.md).

## Thêm skill

[Skills](multi-agent/skills-sql-assistant.md) cung cấp cách cấp kiến thức chuyên biệt theo nhu cầu cho agent khi cần, dùng progressive disclosure. Skill có thể gồm workflow nhiều bước, quy tắc, và quy ước. Bằng cách đặt thông tin này trong một skill, nó không được thêm vào system prompt theo mặc định, đảm bảo token chỉ được dùng khi thông tin từ skill thực sự cần cho một task.

Khi agent khởi động, nó chỉ thấy metadata gọn nhẹ về mỗi skill. Khi một task cần một skill, agent nạp toàn bộ skill file theo nhu cầu.

Tạo một skill file trong thư mục skills:

```
skills/
  pandas-patterns/
    SKILL.md
```

```markdown
---
name: pandas-patterns
description: Common pandas and matplotlib patterns for data analysis and visualization
---

## Data loading
Use `pd.read_csv()` for CSV files. Always check `df.info()` and `df.describe()` first.

## Visualization
Use `matplotlib` for bar charts, `seaborn` for statistical plots.
Save figures with `plt.savefig("output.png", dpi=150, bbox_inches="tight")`.

## Reporting
Write a markdown summary to `report.md` alongside any generated charts.
```

Skill này chứa thông tin về cách việc visualization nên được thực hiện.

Với [`LangSmithSandbox`](https://reference.langchain.com/python/deepagents/backends/langsmith/LangSmithSandbox), đường dẫn skill được phân giải trên filesystem của sandbox, không phải máy local của bạn. Upload thư mục `skills/` local của bạn trước khi cấu hình [`SkillsMiddleware`](https://reference.langchain.com/python/deepagents/middleware/skills/SkillsMiddleware):

```python
from pathlib import Path

skills_dir = (Path(__file__).resolve().parent / "skills").resolve()
skill_files: list[tuple[str, bytes]] = []
for path in sorted(skills_dir.rglob("*")):
    if not path.is_file():
        continue
    rel = path.resolve().relative_to(skills_dir)
    skill_files.append((f"/skills/{rel.as_posix()}", path.read_bytes()))
backend.upload_files(skill_files)
```

Sau đó tạo agent với skill của bạn bằng cách thêm [`SkillsMiddleware`](https://reference.langchain.com/python/deepagents/middleware/skills/SkillsMiddleware):

=== "OpenAI"
    ```python
    from deepagents.middleware import FilesystemMiddleware, SkillsMiddleware, SummarizationMiddleware

    model = "openai:gpt-5.5"

    agent = create_agent(
        model=model,
        tools=[],
        middleware=[
            FilesystemMiddleware(backend=backend),
            SummarizationMiddleware(model=model, backend=backend),
            SkillsMiddleware(backend=backend, sources=["/skills/"]),
        ],
    )
    ```

=== "Anthropic"
    ```python
    from deepagents.middleware import FilesystemMiddleware, SkillsMiddleware, SummarizationMiddleware

    model = "anthropic:claude-sonnet-4-6"

    agent = create_agent(
        model=model,
        tools=[],
        middleware=[
            FilesystemMiddleware(backend=backend),
            SummarizationMiddleware(model=model, backend=backend),
            SkillsMiddleware(backend=backend, sources=["/skills/"]),
        ],
    )
    ```

Bạn có thể thử một prompt như "Analyze sales.csv using our pandas patterns." Agent sẽ nạp skill khi nó cần hướng dẫn về plotting hoặc reporting. Nếu bạn hỏi một câu khác không cần skill, agent sẽ không nạp nó.

## Thêm subagent visualization

Một số task tạo ra output trung gian lớn (bản nháp script, lần chạy thất bại, đọc file) sẽ chiếm chỗ context của agent chính nếu giữ trong một thread. Một [subagent](https://docs.langchain.com/oss/python/deepagents/subagents) chạy trong context window riêng của nó nên supervisor chỉ thấy kết quả cuối cùng, không phải từng tool call dọc đường. Điều đó giữ phần phân tích chính tập trung và để lại chỗ cho các câu hỏi tiếp theo.

Một ví dụ điển hình cho việc dùng subagent là tạo biểu đồ. Việc vẽ biểu đồ thường có nghĩa là lặp lại script Python, cài package, và đọc output lỗi trước khi có được một hình ảnh hoàn chỉnh. Subagent `visualizer` sau đây có thể xử lý công việc đó một cách cô lập trong khi agent chính tiếp tục lập kế hoạch và phân tích. Với [`TodoListMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/todo/TodoListMiddleware), agent chính cũng có thể uỷ quyền công việc biểu đồ đó song song thay vì chặn (block) ở từng biểu đồ.

Cập nhật agent của bạn từ bước 4 bằng cách thêm [`TodoListMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/todo/TodoListMiddleware) và [`SubAgentMiddleware`](https://reference.langchain.com/python/deepagents/middleware/subagents/SubAgentMiddleware):

=== "OpenAI"
    ```python
    from deepagents import SubAgent
    from deepagents.middleware import (
        FilesystemMiddleware,
        SkillsMiddleware,
        SubAgentMiddleware,
        SummarizationMiddleware,
    )
    from langchain.agents.middleware import TodoListMiddleware

    model = "openai:gpt-5.5"

    visualizer: SubAgent = {
        "name": "visualizer",
        "description": "Generates charts and visualizations from data files in the sandbox.",
        "system_prompt": "You are a data visualization specialist. Write Python scripts using matplotlib and seaborn. Save all figures as PNG files.",
        "tools": [],
        "model": model,
    }

    agent = create_agent(
        model=model,
        tools=[],
        middleware=[
            FilesystemMiddleware(backend=backend),
            SummarizationMiddleware(model=model, backend=backend),
            SkillsMiddleware(backend=backend, sources=["/skills/"]),
            TodoListMiddleware(),
            SubAgentMiddleware(backend=backend, subagents=[visualizer]),
        ],
    )
    ```

=== "Anthropic"
    ```python
    from deepagents import SubAgent
    from deepagents.middleware import (
        FilesystemMiddleware,
        SkillsMiddleware,
        SubAgentMiddleware,
        SummarizationMiddleware,
    )
    from langchain.agents.middleware import TodoListMiddleware

    model = "anthropic:claude-sonnet-4-6"

    visualizer: SubAgent = {
        "name": "visualizer",
        "description": "Generates charts and visualizations from data files in the sandbox.",
        "system_prompt": "You are a data visualization specialist. Write Python scripts using matplotlib and seaborn. Save all figures as PNG files.",
        "tools": [],
        "model": model,
    }

    agent = create_agent(
        model=model,
        tools=[],
        middleware=[
            FilesystemMiddleware(backend=backend),
            SummarizationMiddleware(model=model, backend=backend),
            SkillsMiddleware(backend=backend, sources=["/skills/"]),
            TodoListMiddleware(),
            SubAgentMiddleware(backend=backend, subagents=[visualizer]),
        ],
    )
    ```

Thử một prompt như "Analyze sales.csv, then create a bar chart of revenue by product." Agent chính xử lý phân tích và lập kế hoạch, đồng thời uỷ quyền việc tạo biểu đồ cho subagent `visualizer` thông qua tool `task`.

Nếu bạn đã bật tracing ở phần [Thiết lập](#thiet-lap), mở lần chạy trong [LangSmith](https://smith.langchain.com/). Bạn sẽ thấy một lần gọi `task` tới `visualizer`, một sub-run riêng với vòng lặp tool riêng, và một kết quả ngắn gọn được trả về cho supervisor.

## Những gì bạn đã xây dựng

Bạn đã xây một agent tuỳ chỉnh với các middleware sau:

| Middleware | Thêm gì |
| --- | --- |
| [`FilesystemMiddleware`](https://reference.langchain.com/python/deepagents/middleware/filesystem/FilesystemMiddleware) + `LangSmithSandbox` | Filesystem cô lập + tool `execute` |
| [`SummarizationMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/SummarizationMiddleware) | Nén context tự động |
| [`SkillsMiddleware`](https://reference.langchain.com/python/deepagents/middleware/skills/SkillsMiddleware) | Kiến thức chuyên biệt được nạp theo nhu cầu |
| [`TodoListMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/todo/TodoListMiddleware) + [`SubAgentMiddleware`](https://reference.langchain.com/python/deepagents/middleware/subagents/SubAgentMiddleware) | Subagent visualization chạy song song |

Đây chính là nền tảng giống với [`create_deep_agent`](https://reference.langchain.com/python/deepagents/graph/create_deep_agent): được lắp ráp thủ công để bạn kiểm soát chính xác những gì được bao gồm.

Khả năng không dừng lại ở đây: xem [Prebuilt middleware](middleware/built-in.md) để biết danh sách đầy đủ các khả năng có thể kết hợp, và tài liệu tham khảo [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) cho mọi tuỳ chọn cấu hình.

Để làm việc với phiên bản đã lắp ráp sẵn, xem [Customize Deep Agents](https://docs.langchain.com/oss/python/deepagents/customization). Để xem ví dụ phân tích dữ liệu đầy đủ dùng `create_deep_agent`, xem [Data analysis](https://docs.langchain.com/oss/python/deepagents/data-analysis).

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/deep-agent-from-scratch.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
