# Xây dựng một SQL agent tuỳ chỉnh

Trong hướng dẫn này, ta sẽ xây dựng một agent tuỳ chỉnh có thể trả lời các câu hỏi về một cơ sở dữ liệu SQL bằng LangGraph.

LangChain cung cấp sẵn các triển khai [agent](../langchain/agents.md), được xây dựng bằng các primitive của [LangGraph](./overview.md). Nếu cần tuỳ biến sâu hơn, agent có thể được triển khai trực tiếp trong LangGraph. Hướng dẫn này minh hoạ một ví dụ triển khai SQL agent. Để có phần giới thiệu thực tế hơn, xem [xây dựng SQL agent bằng các abstraction cấp cao của LangChain](../langchain/sql-agent.md).

!!! warning ""
    Xây dựng hệ thống hỏi đáp (Q&A) cho cơ sở dữ liệu SQL đòi hỏi thực thi các câu truy vấn SQL do model tạo ra. Việc này tiềm ẩn rủi ro. Hãy đảm bảo quyền kết nối cơ sở dữ liệu của bạn luôn được giới hạn ở phạm vi hẹp nhất có thể theo nhu cầu của agent. Điều này sẽ giảm thiểu, chứ không loại bỏ hoàn toàn, rủi ro khi xây dựng một hệ thống được điều khiển bởi model.

[Agent dựng sẵn](../langchain/sql-agent.md) giúp ta bắt đầu nhanh chóng, nhưng ta phải dựa vào system prompt để giới hạn hành vi của nó, ví dụ, ta chỉ thị agent luôn bắt đầu bằng tool "list tables", và luôn chạy tool kiểm tra truy vấn trước khi thực thi truy vấn đó.

Ta có thể áp đặt mức độ kiểm soát cao hơn trong LangGraph bằng cách tuỳ chỉnh agent. Ở đây, ta triển khai một thiết lập ReAct-agent đơn giản, với các node riêng biệt cho từng lệnh gọi tool cụ thể. Ta sẽ dùng cùng state như agent dựng sẵn.

### Khái niệm

Ta sẽ đề cập đến các khái niệm sau:

* [Tool](../langchain/tools.md) để đọc từ cơ sở dữ liệu SQL
* [Graph API](./graph-api.md) của LangGraph, bao gồm state, node, edge, và conditional edge.
* Các quy trình [human-in-the-loop](./interrupts.md)

## Setup

### Cài đặt

```bash
pip install langchain langgraph
```

### LangSmith

Thiết lập [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langgraph-sql-agent) để quan sát những gì đang diễn ra bên trong chain hoặc agent của bạn. Sau đó thiết lập các biến môi trường sau:

```shell
export LANGSMITH_TRACING="true"
export LANGSMITH_API_KEY="..."
```

## 1. Chọn một LLM

Chọn một model hỗ trợ [tool-calling](https://docs.langchain.com/oss/python/integrations/providers/overview):

=== "OpenAI"
    👉 Đọc [tài liệu tích hợp chat model OpenAI](https://docs.langchain.com/oss/python/integrations/chat/openai/)

    ```bash pip
    pip install -U "langchain[openai]"
    ```

    ```bash uv
    uv add "langchain[openai]"
    ```

    ```python init_chat_model
    import os
    from langchain.chat_models import init_chat_model

    os.environ["OPENAI_API_KEY"] = "sk-..."

    model = init_chat_model("gpt-5.5")
    ```

    ```python Model Class
    import os
    from langchain_openai import ChatOpenAI

    os.environ["OPENAI_API_KEY"] = "sk-..."

    model = ChatOpenAI(model="gpt-5.5")
    ```

=== "Anthropic"
    👉 Đọc [tài liệu tích hợp chat model Anthropic](https://docs.langchain.com/oss/python/integrations/chat/anthropic/)

    ```bash pip
    pip install -U "langchain[anthropic]"
    ```

    ```bash uv
    uv add "langchain[anthropic]"
    ```

    ```python init_chat_model
    import os
    from langchain.chat_models import init_chat_model

    os.environ["ANTHROPIC_API_KEY"] = "sk-..."

    model = init_chat_model("claude-sonnet-4-6")
    ```

    ```python Model Class
    import os
    from langchain_anthropic import ChatAnthropic

    os.environ["ANTHROPIC_API_KEY"] = "sk-..."

    model = ChatAnthropic(model="claude-sonnet-4-6")
    ```

=== "Azure"
    👉 Đọc [tài liệu tích hợp chat model Azure](https://docs.langchain.com/oss/python/integrations/chat/azure_chat_openai/)

    ```bash pip
    pip install -U "langchain[openai]"
    ```

    ```bash uv
    uv add "langchain[openai]"
    ```

    ```python init_chat_model
    import os
    from langchain.chat_models import init_chat_model

    os.environ["AZURE_OPENAI_API_KEY"] = "..."
    os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
    os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

    model = init_chat_model(
        "azure_openai:gpt-5.5",
        azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"],
    )
    ```

    ```python Model Class
    import os
    from langchain_openai import AzureChatOpenAI

    os.environ["AZURE_OPENAI_API_KEY"] = "..."
    os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
    os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

    model = AzureChatOpenAI(
        model="gpt-5.5",
        azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"]
    )
    ```

=== "Google Gemini"
    👉 Đọc [tài liệu tích hợp chat model Google GenAI](https://docs.langchain.com/oss/python/integrations/chat/google_generative_ai/)

    ```bash pip
    pip install -U "langchain[google-genai]"
    ```

    ```bash uv
    uv add "langchain[google-genai]"
    ```

    ```python init_chat_model
    import os
    from langchain.chat_models import init_chat_model

    os.environ["GOOGLE_API_KEY"] = "..."

    model = init_chat_model("google_genai:gemini-2.5-flash-lite")
    ```

    ```python Model Class
    import os
    from langchain_google_genai import ChatGoogleGenerativeAI

    os.environ["GOOGLE_API_KEY"] = "..."

    model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
    ```

=== "AWS Bedrock"
    👉 Đọc [tài liệu tích hợp chat model AWS Bedrock](https://docs.langchain.com/oss/python/integrations/chat/bedrock/)

    ```bash pip
    pip install -U "langchain[aws]"
    ```

    ```bash uv
    uv add "langchain[aws]"
    ```

    ```python init_chat_model
    from langchain.chat_models import init_chat_model

    # Làm theo các bước tại đây để cấu hình credential của bạn:
    # https://docs.aws.amazon.com/bedrock/latest/userguide/getting-started.html

    model = init_chat_model(
        "us.anthropic.claude-sonnet-4-6",
        model_provider="bedrock_converse",
    )
    ```

    ```python Model Class
    from langchain_aws import ChatBedrock

    model = ChatBedrock(model="us.anthropic.claude-sonnet-4-6")
    ```

=== "HuggingFace"
    👉 Đọc [tài liệu tích hợp chat model HuggingFace](https://docs.langchain.com/oss/python/integrations/chat/huggingface/)

    ```bash pip
    pip install -U "langchain[huggingface]"
    ```

    ```bash uv
    uv add "langchain[huggingface]"
    ```

    ```python init_chat_model
    import os
    from langchain.chat_models import init_chat_model

    os.environ["HUGGINGFACEHUB_API_TOKEN"] = "hf_..."

    model = init_chat_model(
        "microsoft/Phi-3-mini-4k-instruct",
        model_provider="huggingface",
        temperature=0.7,
        max_tokens=1024,
    )
    ```

    ```python Model Class
    import os
    from langchain_huggingface import ChatHuggingFace, HuggingFaceEndpoint

    os.environ["HUGGINGFACEHUB_API_TOKEN"] = "hf_..."

    llm = HuggingFaceEndpoint(
        repo_id="microsoft/Phi-3-mini-4k-instruct",
        temperature=0.7,
        max_length=1024,
    )
    model = ChatHuggingFace(llm=llm)
    ```

=== "OpenRouter"
    👉 Đọc [tài liệu tích hợp chat model OpenRouter](https://docs.langchain.com/oss/python/integrations/chat/openrouter/)

    ```bash pip
    pip install -U "langchain-openrouter"
    ```

    ```bash uv
    uv add "langchain-openrouter"
    ```

    ```python init_chat_model
    import os
    from langchain.chat_models import init_chat_model

    os.environ["OPENROUTER_API_KEY"] = "sk-..."

    model = init_chat_model(
        "auto",
        model_provider="openrouter",
    )
    ```

    ```python Model Class
    import os
    from langchain_openrouter import ChatOpenRouter

    os.environ["OPENROUTER_API_KEY"] = "sk-..."

    model = ChatOpenRouter(model="auto")
    ```

Output hiển thị trong các ví dụ bên dưới dùng OpenAI.

## 2. Cấu hình cơ sở dữ liệu

Bạn sẽ tạo một [cơ sở dữ liệu SQLite](https://www.sqlitetutorial.net/sqlite-sample-database/) cho hướng dẫn này. SQLite là một cơ sở dữ liệu nhẹ, dễ thiết lập và sử dụng. Ta sẽ nạp cơ sở dữ liệu `chinook`, một cơ sở dữ liệu mẫu đại diện cho một cửa hàng media kỹ thuật số.

Để thuận tiện, chúng tôi đã lưu trữ sẵn cơ sở dữ liệu này (`Chinook.db`) trên một GCS bucket công khai.

```python
import pathlib
import requests

url = "https://storage.googleapis.com/benchmarks-artifacts/chinook/Chinook.db"
local_path = pathlib.Path("Chinook.db")

if local_path.exists():
    print(f"{local_path} already exists, skipping download.")
else:
    response = requests.get(url, timeout=60)
    if response.status_code == 200:
        local_path.write_bytes(response.content)
        print(f"File downloaded and saved as {local_path}")
    else:
        print(f"Failed to download the file. Status code: {response.status_code}")
```

Ta sẽ dùng module `sqlite3` có sẵn của Python để tương tác với cơ sở dữ liệu:

```python
import sqlite3

con = sqlite3.connect("Chinook.db")
cursor = con.cursor()

cursor.execute("SELECT name FROM sqlite_master WHERE type='table';")
tables = [row[0] for row in cursor.fetchall() if not row[0].startswith("sqlite_")]

print("Dialect: sqlite")
print(f"Available tables: {tables}")

cursor.execute("SELECT * FROM Artist LIMIT 5;")
print(f"Sample output: {cursor.fetchall()}")
con.close()
```

```
Dialect: sqlite
Available tables: ['Album', 'Artist', 'Customer', 'Employee', 'Genre', 'Invoice', 'InvoiceLine', 'MediaType', 'Playlist', 'PlaylistTrack', 'Track']
Sample output: [(1, 'AC/DC'), (2, 'Accept'), (3, 'Aerosmith'), (4, 'Alanis Morissette'), (5, 'Alice In Chains')]
```

## 3. Thêm tool để tương tác với cơ sở dữ liệu

!!! warning ""
    Các tool cơ sở dữ liệu dưới đây chỉ là các wrapper tối giản dùng cho mục đích minh hoạ. Chúng không được thiết kế để an toàn hay dùng trong production. Hãy dùng quyền cơ sở dữ liệu giới hạn hẹp và bổ sung validation riêng cho ứng dụng trước khi thực thi SQL do model tạo ra.

Ta có thể triển khai các [tool](../langchain/tools.md) cơ sở dữ liệu dưới dạng các wrapper mỏng bằng decorator `@tool` từ `langchain.tools`:

```python
import sqlite3
from langchain.tools import tool

# Dưới đây là các tool tối giản dùng cho mục đích minh hoạ.


@tool
def sql_db_list_tables() -> str:
    """Input is an empty string, output is a comma-separated list of tables in the database."""
    con = sqlite3.connect("Chinook.db")
    try:
        cursor = con.cursor()
        cursor.execute("SELECT name FROM sqlite_master WHERE type='table';")
        tables = [
            row[0]
            for row in cursor.fetchall()
            if not row[0].startswith("sqlite_")
        ]
        return ", ".join(tables)
    finally:
        con.close()


@tool
def sql_db_schema(table_names: str) -> str:
    """Input to this tool is a comma-separated list of tables, output is the schema and sample rows for those tables.
    Be sure that the tables actually exist by calling sql_db_list_tables first!
    Example Input: table1, table2, table3"""
    con = sqlite3.connect("Chinook.db")
    try:
        cursor = con.cursor()
        cursor.execute("SELECT name FROM sqlite_master WHERE type='table';")
        valid_tables = {
            row[0] for row in cursor.fetchall() if not row[0].startswith("sqlite_")
        }
        results = []
        for table in table_names.split(","):
            table = table.strip()
            if table not in valid_tables:
                results.append(
                    f"Error: table_names {{{table!r}}} not found in database"
                )
                continue
            cursor.execute(
                "SELECT sql FROM sqlite_master WHERE type='table' AND name=?;",
                (table,),
            )
            schema_row = cursor.fetchone()
            if schema_row:
                results.append(schema_row[0])
                try:
                    quoted_table = '"' + table.replace('"', '""') + '"'
                    cursor.execute(f"SELECT * FROM {quoted_table} LIMIT 3;")
                    rows = cursor.fetchall()
                    if rows:
                        col_names = [description[0] for description in cursor.description]
                        results.append(
                            f"/*\n3 rows from {table} table:\n"
                            + "\t".join(col_names)
                            + "\n"
                            + "\n".join(
                                "\t".join(str(x) for x in row) for row in rows
                            )
                            + "\n*/"
                        )
                except Exception as e:
                    results.append(f"Error fetching sample rows: {e}")
        return "\n\n".join(results)
    finally:
        con.close()


@tool
def sql_db_query(query: str) -> str:
    """Input to this tool is a detailed and correct SQL query, output is a result from the database.
    If the query is not correct, an error message will be returned.
    If an error is returned, rewrite the query, check the query, and try again.
    If you encounter an issue with Unknown column 'xxxx' in 'field list', use sql_db_schema to query the correct table fields."""
    con = sqlite3.connect("Chinook.db")
    try:
        cursor = con.cursor()
        cursor.execute(query)
        res = cursor.fetchall()
        return str(res)
    except Exception as e:
        return f"Error: {e}"
    finally:
        con.close()


tools = [sql_db_list_tables, sql_db_schema, sql_db_query]

# Use a distinct loop variable so it does not shadow the `tool` decorator,
# which is reused later to wrap the query tool for human review.
for t in tools:
    print(f"{t.name}: {t.description}\n")
```

```
sql_db_list_tables: Input is an empty string, output is a comma-separated list of tables in the database.

sql_db_schema: Input to this tool is a comma-separated list of tables, output is the schema and sample rows for those tables.
    Be sure that the tables actually exist by calling sql_db_list_tables first!
    Example Input: table1, table2, table3

sql_db_query: Input to this tool is a detailed and correct SQL query, output is a result from the database.
    If the query is not correct, an error message will be returned.
    If an error is returned, rewrite the query, check the query, and try again.
    If you encounter an issue with Unknown column 'xxxx' in 'field list', use sql_db_schema to query the correct table fields.
```

## 4. Định nghĩa các bước của ứng dụng

Ta xây dựng các node riêng biệt cho các bước sau:

* Liệt kê bảng trong DB
* Gọi tool "get schema"
* Tạo truy vấn
* Kiểm tra truy vấn

Đặt các bước này vào các node riêng cho phép ta (1) buộc thực hiện tool-call khi cần, và (2) tuỳ chỉnh prompt gắn với từng bước.

```python
from typing import Literal

from langchain.messages import AIMessage
from langchain_core.runnables import RunnableConfig
from langgraph.graph import END, START, MessagesState, StateGraph
from langgraph.prebuilt import ToolNode

get_schema_tool = next(tool for tool in tools if tool.name == "sql_db_schema")
get_schema_node = ToolNode([get_schema_tool], name="get_schema")

run_query_tool = next(tool for tool in tools if tool.name == "sql_db_query")
run_query_node = ToolNode([run_query_tool], name="run_query")


# Ví dụ: tạo một tool call được định trước
def list_tables(state: MessagesState):
    tool_call = {
        "name": "sql_db_list_tables",
        "args": {},
        "id": "abc123",
        "type": "tool_call",
    }
    tool_call_message = AIMessage(content="", tool_calls=[tool_call])

    list_tables_tool = next(tool for tool in tools if tool.name == "sql_db_list_tables")
    tool_message = list_tables_tool.invoke(tool_call)
    response = AIMessage(f"Available tables: {tool_message.content}")

    return {"messages": [tool_call_message, tool_message, response]}


# Ví dụ: buộc model tạo một tool call
def call_get_schema(state: MessagesState):
    # Lưu ý LangChain yêu cầu mọi model phải chấp nhận `tool_choice="any"`
    # cũng như `tool_choice=<string name of tool>`.
    llm_with_tools = model.bind_tools([get_schema_tool], tool_choice="any")
    response = llm_with_tools.invoke(state["messages"])

    return {"messages": [response]}


generate_query_system_prompt = """
You are an agent designed to interact with a SQL database.
Given an input question, create a syntactically correct {dialect} query to run,
then look at the results of the query and return the answer. Unless the user
specifies a specific number of examples they wish to obtain, always limit your
query to at most {top_k} results.

You can order the results by a relevant column to return the most interesting
examples in the database. Never query for all the columns from a specific table,
only ask for the relevant columns given the question.

DO NOT make any DML statements (INSERT, UPDATE, DELETE, DROP etc.) to the database.
""".format(
    dialect="sqlite",
    top_k=5,
)


def generate_query(state: MessagesState):
    system_message = {
        "role": "system",
        "content": generate_query_system_prompt,
    }
    # Ta không ép buộc tool call ở đây, để model có thể
    # phản hồi tự nhiên khi đã có được lời giải.
    llm_with_tools = model.bind_tools([run_query_tool])
    response = llm_with_tools.invoke([system_message] + state["messages"])

    return {"messages": [response]}


check_query_system_prompt = """
You are a SQL expert with a strong attention to detail.
Double check the {dialect} query for common mistakes, including:
- Using NOT IN with NULL values
- Using UNION when UNION ALL should have been used
- Using BETWEEN for exclusive ranges
- Data type mismatch in predicates
- Properly quoting identifiers
- Using the correct number of arguments for functions
- Casting to the correct data type
- Using the proper columns for joins

If there are any of the above mistakes, rewrite the query. If there are no mistakes,
just reproduce the original query.

You will call the appropriate tool to execute the query after running this check.
""".format(dialect="sqlite")


def check_query(state: MessagesState):
    system_message = {
        "role": "system",
        "content": check_query_system_prompt,
    }

    # Tạo một tin nhắn người dùng nhân tạo để kiểm tra
    tool_call = state["messages"][-1].tool_calls[0]
    user_message = {"role": "user", "content": tool_call["args"]["query"]}
    llm_with_tools = model.bind_tools([run_query_tool], tool_choice="any")
    response = llm_with_tools.invoke([system_message, user_message])
    response.id = state["messages"][-1].id

    return {"messages": [response]}
```

## 5. Triển khai agent

Giờ ta có thể ghép các bước này thành một workflow bằng [Graph API](./graph-api.md). Ta định nghĩa một [conditional edge](./graph-api.md#conditional-edges) tại bước tạo truy vấn, sẽ định tuyến tới bước kiểm tra truy vấn nếu có truy vấn được tạo ra, hoặc kết thúc nếu không có tool call nào, tức LLM đã đưa ra câu trả lời cho truy vấn.

```python
def should_continue(state: MessagesState) -> Literal[END, "check_query"]:
    messages = state["messages"]
    last_message = messages[-1]
    if not last_message.tool_calls:
        return END
    else:
        return "check_query"


builder = StateGraph(MessagesState)
builder.add_node(list_tables)
builder.add_node(call_get_schema)
builder.add_node(get_schema_node, "get_schema")
builder.add_node(generate_query)
builder.add_node(check_query)
builder.add_node(run_query_node, "run_query")

builder.add_edge(START, "list_tables")
builder.add_edge("list_tables", "call_get_schema")
builder.add_edge("call_get_schema", "get_schema")
builder.add_edge("get_schema", "generate_query")
builder.add_conditional_edges(
    "generate_query",
    should_continue,
)
builder.add_edge("check_query", "run_query")
builder.add_edge("run_query", "generate_query")

agent = builder.compile()
```

Ta trực quan hoá ứng dụng bên dưới:

```python
import pathlib

pathlib.Path("graph.png").write_bytes(agent.get_graph().draw_mermaid_png())
```

<img src="https://mintcdn.com/langchain-5e9cc07a/aAi4RLdXQAh8fThS/oss/images/sql-agent-langgraph.png?fit=max&auto=format&n=aAi4RLdXQAh8fThS&q=85&s=1ddd4aae369fb8c143edaccb0a09c81f" alt="SQL agent graph" style="height: 800px" width="308" height="645" data-path="oss/images/sql-agent-langgraph.png" />

Giờ ta có thể gọi graph:

```python
question = "Which genre on average has the longest tracks?"

stream = agent.stream_events(
    {"messages": [{"role": "user", "content": question}]},
    version="v3",
)
for message in stream.messages:
    for token in message.text:
        print(token, end="", flush=True)

final_state = stream.output
```

```
================================ Human Message =================================

Which genre on average has the longest tracks?
================================== Ai Message ==================================

Available tables: Album, Artist, Customer, Employee, Genre, Invoice, InvoiceLine, MediaType, Playlist, PlaylistTrack, Track
================================== Ai Message ==================================
Tool Calls:
  sql_db_schema (call_yzje0tj7JK3TEzDx4QnRR3lL)
 Call ID: call_yzje0tj7JK3TEzDx4QnRR3lL
  Args:
    table_names: Genre, Track
================================= Tool Message =================================
Name: sql_db_schema


CREATE TABLE "Genre" (
	"GenreId" INTEGER NOT NULL,
	"Name" NVARCHAR(120),
	PRIMARY KEY ("GenreId")
)

/*
3 rows from Genre table:
GenreId	Name
1	Rock
2	Jazz
3	Metal
*/


CREATE TABLE "Track" (
	"TrackId" INTEGER NOT NULL,
	"Name" NVARCHAR(200) NOT NULL,
	"AlbumId" INTEGER,
	"MediaTypeId" INTEGER NOT NULL,
	"GenreId" INTEGER,
	"Composer" NVARCHAR(220),
	"Milliseconds" INTEGER NOT NULL,
	"Bytes" INTEGER,
	"UnitPrice" NUMERIC(10, 2) NOT NULL,
	PRIMARY KEY ("TrackId"),
	FOREIGN KEY("MediaTypeId") REFERENCES "MediaType" ("MediaTypeId"),
	FOREIGN KEY("GenreId") REFERENCES "Genre" ("GenreId"),
	FOREIGN KEY("AlbumId") REFERENCES "Album" ("AlbumId")
)

/*
3 rows from Track table:
TrackId	Name	AlbumId	MediaTypeId	GenreId	Composer	Milliseconds	Bytes	UnitPrice
1	For Those About To Rock (We Salute You)	1	1	1	Angus Young, Malcolm Young, Brian Johnson	343719	11170334	0.99
2	Balls to the Wall	2	2	1	U. Dirkschneider, W. Hoffmann, H. Frank, P. Baltes, S. Kaufmann, G. Hoffmann	342562	5510424	0.99
3	Fast As a Shark	3	2	1	F. Baltes, S. Kaufman, U. Dirkscneider & W. Hoffman	230619	3990994	0.99
*/
================================== Ai Message ==================================
Tool Calls:
  sql_db_query (call_cb9ApLfZLSq7CWg6jd0im90b)
 Call ID: call_cb9ApLfZLSq7CWg6jd0im90b
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgMilliseconds FROM Track JOIN Genre ON Track.GenreId = Genre.GenreId GROUP BY Genre.GenreId ORDER BY AvgMilliseconds DESC LIMIT 5;
================================== Ai Message ==================================
Tool Calls:
  sql_db_query (call_DMVALfnQ4kJsuF3Yl6jxbeAU)
 Call ID: call_DMVALfnQ4kJsuF3Yl6jxbeAU
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgMilliseconds FROM Track JOIN Genre ON Track.GenreId = Genre.GenreId GROUP BY Genre.GenreId ORDER BY AvgMilliseconds DESC LIMIT 5;
================================= Tool Message =================================
Name: sql_db_query

[('Sci Fi & Fantasy', 2911783.0384615385), ('Science Fiction', 2625549.076923077), ('Drama', 2575283.78125), ('TV Shows', 2145041.0215053763), ('Comedy', 1585263.705882353)]
================================== Ai Message ==================================

The genre with the longest tracks on average is "Sci Fi & Fantasy," with an average track length of approximately 2,911,783 milliseconds. Other genres with relatively long tracks include "Science Fiction," "Drama," "TV Shows," and "Comedy."
```

!!! tip ""
    Xem [LangSmith trace](https://smith.langchain.com/public/94b8c9ac-12f7-4692-8706-836a1f30f1ea/r) cho lượt chạy trên.

## 6. Triển khai kiểm duyệt human-in-the-loop

Sẽ là thận trọng nếu kiểm tra các truy vấn SQL của agent trước khi chúng được thực thi, để tránh các hành động không mong muốn hoặc thiếu hiệu quả.

Ở đây ta tận dụng các tính năng [human-in-the-loop](./interrupts.md) của LangGraph để tạm dừng lượt chạy trước khi thực thi một truy vấn SQL và chờ con người xem xét. Bằng cách dùng [lớp persistence](./persistence.md) của LangGraph, ta có thể tạm dừng lượt chạy vô thời hạn (hoặc ít nhất là chừng nào lớp persistence còn hoạt động).

Hãy bọc tool `sql_db_query` trong một node nhận input từ con người. Ta có thể triển khai điều này bằng hàm [interrupt](./interrupts.md). Bên dưới, ta cho phép input để chấp thuận tool call, chỉnh sửa tham số của nó, hoặc cung cấp phản hồi từ người dùng.

```python
from langchain.tools import tool
from langgraph.types import interrupt
from langchain_core.runnables import RunnableConfig


@tool(
    run_query_tool.name,
    description=run_query_tool.description,
    args_schema=run_query_tool.args_schema,
)
def run_query_tool_with_interrupt(config: RunnableConfig, **tool_input):
    request = {
        "action": run_query_tool.name,
        "args": tool_input,
        "description": "Please review the tool call",
    }
    response = interrupt([request])  # [!code highlight]
    # chấp thuận tool call
    if response["type"] == "accept":
        tool_response = run_query_tool.invoke(tool_input, config)
    # cập nhật tham số tool call
    elif response["type"] == "edit":
        tool_input = response["args"]["args"]
        tool_response = run_query_tool.invoke(tool_input, config)
    # phản hồi lại LLM bằng feedback từ người dùng
    elif response["type"] == "response":
        user_feedback = response["args"]
        tool_response = user_feedback
    else:
        raise ValueError(f"Unsupported interrupt response type: {response['type']}")

    return tool_response


# Định nghĩa lại tool node để dùng phiên bản có interrupt
run_query_node = ToolNode([run_query_tool_with_interrupt], name="run_query")  # [!code highlight]
```

!!! note ""
    Cách triển khai trên tuân theo [ví dụ interrupt cho tool](./interrupts.md#interrupts-in-tools) trong hướng dẫn [human-in-the-loop](./interrupts.md) tổng quát hơn. Tham khảo hướng dẫn đó để biết chi tiết và các lựa chọn khác.

Giờ hãy ghép lại graph. Ta sẽ thay việc kiểm tra bằng chương trình bằng kiểm duyệt của con người. Lưu ý ta giờ đã bao gồm một [checkpointer](./persistence.md); đây là yêu cầu bắt buộc để tạm dừng và tiếp tục lượt chạy.

```python
from langgraph.checkpoint.memory import InMemorySaver

def should_continue(state: MessagesState) -> Literal[END, "run_query"]:
    messages = state["messages"]
    last_message = messages[-1]
    if not last_message.tool_calls:
        return END
    else:
        return "run_query"

builder = StateGraph(MessagesState)
builder.add_node(list_tables)
builder.add_node(call_get_schema)
builder.add_node(get_schema_node, "get_schema")
builder.add_node(generate_query)
builder.add_node(run_query_node, "run_query")

builder.add_edge(START, "list_tables")
builder.add_edge("list_tables", "call_get_schema")
builder.add_edge("call_get_schema", "get_schema")
builder.add_edge("get_schema", "generate_query")
builder.add_conditional_edges(
    "generate_query",
    should_continue,
)
builder.add_edge("run_query", "generate_query")

checkpointer = InMemorySaver()  # [!code highlight]
agent = builder.compile(checkpointer=checkpointer)  # [!code highlight]
```

Ta có thể gọi graph như trước. Lần này, việc thực thi bị gián đoạn (interrupted):

```python
question = "Which genre on average has the longest tracks?"

stream = agent.stream_events(
    {"messages": [{"role": "user", "content": question}]},
    config,
    version="v3",
)
for message in stream.messages:
    for token in message.text:
        print(token, end="", flush=True)
if stream.interrupted:
    action = stream.interrupts[0]
    print("INTERRUPTED:")
    for request in action.value:
        print(json.dumps(request, indent=2))
```

```
...

INTERRUPTED:
{
  "action": "sql_db_query",
  "args": {
    "query": "SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgLength FROM Track JOIN Genre ON Track.GenreId = Genre.GenreId GROUP BY Genre.Name ORDER BY AvgLength DESC LIMIT 5;"
  },
  "description": "Please review the tool call"
}
```

Ta có thể chấp thuận hoặc chỉnh sửa tool call bằng [Command](./use-graph-api.md):

```python
from langgraph.types import Command

stream = agent.stream_events(
    Command(resume={"type": "accept"}),
    # Command(resume={"type": "edit", "args": {"query": "..."}}),
    config,
    version="v3",
)
for message in stream.messages:
    for token in message.text:
        print(token, end="", flush=True)
if stream.interrupted:
    action = stream.interrupts[0]
    print("INTERRUPTED:")
    for request in action.value:
        print(json.dumps(request, indent=2))
```

```
================================== Ai Message ==================================
Tool Calls:
  sql_db_query (call_t4yXkD6shwdTPuelXEmY3sAY)
 Call ID: call_t4yXkD6shwdTPuelXEmY3sAY
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgLength FROM Track JOIN Genre ON Track.GenreId = Genre.GenreId GROUP BY Genre.Name ORDER BY AvgLength DESC LIMIT 5;
================================= Tool Message =================================
Name: sql_db_query

[('Sci Fi & Fantasy', 2911783.0384615385), ('Science Fiction', 2625549.076923077), ('Drama', 2575283.78125), ('TV Shows', 2145041.0215053763), ('Comedy', 1585263.705882353)]
================================== Ai Message ==================================

The genre with the longest average track length is "Sci Fi & Fantasy" with an average length of about 2,911,783 milliseconds. Other genres with long average track lengths include "Science Fiction," "Drama," "TV Shows," and "Comedy."
```

Tham khảo [hướng dẫn human-in-the-loop](./interrupts.md) để biết chi tiết.

## Bước tiếp theo

Xem hướng dẫn [Evaluate a graph](https://docs.langchain.com/langsmith/evaluate-graph) để đánh giá các ứng dụng LangGraph, bao gồm cả SQL agent như agent này, bằng LangSmith.

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/sql-agent.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
